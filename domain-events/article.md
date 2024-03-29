# Handling Side Effects in Rails with Domain Events

- When an order is confirmed, send an SMS to the user.
- When a record is created, update the ElasticSearch index.
- When a claim is submitted, update the policyholder's risk profile.
  
In these examples, a change in an application's domain model triggers some reaction or side effect in another part of the application.  Although quite common and apparently innocuous, side effects[^1] like these can pose a software design challenge.

Side effects often require interactions between disparate parts of an application.  If we naively couple these parts together, we might create a tangled graph of dependencies as our application grows in complexity, making it difficult to understand, test, and change.  Furthermore, we can end up with domain models (the core of our application) that depend directly on peripheral implementation details, like a particular SMS Service's API.

Suppose, for example, that we have a forum application.  It's quite complicated, so we've attempted to decompose it into a few logical subsystems or modules[^2]: one for the forum part itself, which handles user posts and comments, one for content moderation, and one for notifications. The idea here is to create boundaries with clear interfaces so that the internals of these modules can evolve and change independently.  Let's look at a particular case where a domain change triggers a side effect.

When a user is banned, we want to do three things:

- Update the user status to banned.
- Create a moderation case record concerning the user ban.
- Notify the user.

We also want the user status update and the case creation to occur in a database transaction.  We don't want to ban a user without creating a moderation case or create a moderation case without updating the user status; if one operation fails, the other should be rolled back.

As a first pass, we might implement `User#ban!` like this, calling the moderation case and notification creation methods directly from the model.

```ruby
class User < ApplicationRecord
  # omitting associations and validations...
  enum :status, %i[inactive active banned]

  def ban!(reason)
    return true if banned?

    transaction do
      banned!
      Moderation::Case.create_user_banned!(id, reason:)
    end
    Notification::Content.create_for_user!(id, title: 'Banned',
                                               body: "You've been banned due to #{reason}")
    true
  end
end
```

The `ban!` method is self-contained and easy to follow; we can clearly see everything that happens when a user is banned.  However, `User` now depends directly on `Moderation::Case` and `Notification::Content`.  This doesn't look so bad with a single method.  But, consider all of the operations we want `User` to be able to perform (activate, inactivate, remove ban, etc.) that will also have similar `Moderation` and `Notification` side effects.  Changes to any of the `Moderation` or `Notification` classes or methods that `User` relies on will require us to dig into these `User` methods (and probably their tests) and modify them accordingly.  Notice that this breaks down the module barriers that we nominally set up.  We can't modify the relevant `Moderation` or `Notification` classes without considering how they might affect `User`.

Here's a common alternative: Instead of calling side effect code directly in the `User` model and thus coupling it to those classes, we'll push the side effect code out into a Service Object (or alternatively a Controller method).  `User#ban!` will only be responsible for updating the internal state of the model and the Service Object, `BanUser`, will do the rest.

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # ...
  enum :status, %i[inactive active banned]

  def ban!
    banned!
  end
end

# app/services/ban_user.rb
class BanUser
  attr_reader :user, :reason

  def self.call(...)
    new(...).call
  end

  def initialize(user, reason)
    @user = user
    @reason = reason
  end

  def call
    return true if user.banned?

    User.transaction do
      user.ban!
      Moderation::Case.create_user_banned!(id, reason:)
    end
    Notification::Content.create_for_user!(id, title: 'Banned',
                                               body: "You've been banned due to #{reason}")
    true
  end
end
```

This avoids the direct coupling between `User` and `Modification::Case` and `Notification::Content`, but it still has a couple of issues.

One of the nice things about the previous approach, where everything was in the model, is that the model change and the code to trigger its side effects are co-located.  `User#ban!` is the obvious method to call for banning a user.  Whenever it is called, the desired side effects are also triggered.  You would have to go out of your way to ignore that method and update `User#status` directly.  And we could even make that impossible by defining private overrides of the stock ActiveRecord enum methods.

However, with the Service Object, the model change and side effect trigger are one step removed.  Anywhere in the codebase where we want to ban a user, we would have to know that `User#ban!` does not perform the entire operation and that we must invoke the intermediate `BanUser` class to ensure that the side effects are triggered.  This is less obvious and easy to miss, especially if this codebase does not use Service Objects consistently for all operations.[^3]

Another issue with the Service Object approach is that it violates the [Open-Closed Principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle) with respect to side effects.  Suppose that a few weeks after `BanUser` is released, we decide that an additional notification to forum admins should be sent when a user is banned.  This would require modifying `BanUser#call`.  This code has already been tested and is running in production.  Modifying it risks introducing bugs into working code.  It would be preferable if we could add or remove side effects without touching the code that performs the main operation.  As we'll see, domain events would allow us to do that.

## Domain Events

TODO: describe domain events

Let's look at how domain events would be incorporated into our user banning example.  Ignore the `Event::` base classes for now (e.g., `Event::ApplicationEvent`); I'll explain implementations in the [next section](#implementing-domain-events-in-rails).

We have our domain event class, `UserBanned`, that holds the attributes necessary to describe the particular event instance.

```ruby
module Events
  class UserBanned < Event::ApplicationEvent
    attr_reader :user_id, :reason

    def initialize(user_id, reason)
      @user_id = user_id
      @reason = reason
      super
    end
  end
end
```

Back in our `User#ban!` method, we've replaced any side effect code with a call to `add_event`, passing in a `UserBanned` instance.  `add_event` comes from the `Event::Emitter` concern that we've now included.  It stores the events in an array on this model instance.

```ruby
class User < ApplicationRecord
  include Event::Emitter
  # ...
  enum :status, %i[inactive active banned]

  def ban!(reason)
    return true if banned?

    add_event(Events::UserBanned.new(id, reason))
    banned!
    true
  end
end
```

Notice that we added the event prior to calling `banned!`, which will save the model and then commit the current transaction.  `Event::Emitter` registers `after_save` and `after_commit` ActiveRecord callbacks that will publish the events stored on a model instance to the appropriate handlers.  Whether a handler receives an event before or after the transaction committed depends on how it is configured, as we'll see below.

This pattern of adding events to a model and then dispatching them later on ORM lifecycle hooks is called 'the deferred approach' (see [Jimmy Bogard's blog post](https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/); cf. [here](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation#the-deferred-approach-to-raise-and-dispatch-events))  This is in contrast to an immediate approach in which events are raised and dispatched immediately in a single method call.

The benefits of the deferred approach are:

TODO


```ruby
# app/models/moderation/events/create_case_on_user_banned.rb
module Moderation
  module Events
    class CreateCaseOnUserBanned < Event::ApplicationEventHandler
      handles ::Events::UserBanned, on: :before_commit

      def handle(event)
        Case.create_user_banned!(event.user_id, reason: event.reason)
      end
    end
  end
end

# app/models/notification/events/notify_on_user_banned.rb
module Notification
  module Events
    class NotifyOnUserBanned < Event::ApplicationEventHandler
      handles ::Events::UserBanned

      def handle(event)
        Content.create_for_user!(event.user_id, title: 'Banned', body: "You've been banned.")
      end
    end
  end
end
```

## Implementing Domain Events in Rails

I'm going describe one option for implementing the behavior just described.  It is around 150 lines of code and uses one gem, which Rails already uses, namely [Concurrent Ruby](https://github.com/ruby-concurrency/concurrent-ruby).  If you need other features, like persisting the events to a database, you may want to incorporate a gem like [Rails Event Store](https://railseventstore.org/).

TODO

Event
Registry
Publisher
Model
Handlers

Handlers --(Event class)--> Registry
Model --(Event)--> Publisher <--(Handlers)-- Registry

## Tradeoffs

TODO

[^1]: TODO: FOOTNOTE DISTINGUISHING FROM FUNCTIONAL 'SIDE EFFECT'
[^2]: To avoid confusion: I don't mean Ruby modules here. I mean self-contained logical units with well-defined boundaries.
[^3]: We could remove the `User#ban!` method and have `BanUser` update the `User`'s status directly to potentially reduce the confusion.  Apply this strategy generally and we've removed any real behavior from our domain model.  In other words, we have an [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html).

- https://lostechies.com/jimmybogard/2014/05/13/a-better-domain-events-pattern/
- https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation