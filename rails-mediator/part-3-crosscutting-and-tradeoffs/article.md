# Rails on Mediator Part 3: Cross-cutting Concerns and Tradeoffs

Are you having trouble managing the complexity of your Ruby on Rails application?  Are your models tangled together and difficult to change?  Do your controllers contain a lot of complex logic making them difficult to understand and test?  In this 3-part series, I'll suggest some ways that using a mediator to organize parts of your application could help manage this complexity.  Be sure to read [Part 1](../part-1-intro-and-controllers/article.md) where I introduced the mediator pattern and showed how you can use it to make your controllers thinner.  In [Part 2](../part-2-domain-events/article.md) I discussed decoupling your models with domain events. In this final part, we'll learn how to implement cross-cutting concerns without adding more code to your controllers.  We'll also consider some of the general trade-offs of using a mediator to organize Rails applications.

## Cross-cutting concerns

Cross-cutting concerns are parts of an application that are used across feature modules.  Examples include logging, caching, transaction management, and authorization.  Let's consider authorization and see if a mediator can help us improve on the standard way that it's handled in Rails.

### An Example: Authorization

We're going to use the [Pundit gem](https://github.com/varvet/pundit) to help us with authorization and we'll consider a single operation: creating a post.  Our rule is that a user can create a post as long as they're not banned.  The policy class will look like this:

```ruby
class PostPolicy < ApplicationPolicy
  def create?
    !user.banned?
  end
end
```

TODO (WIP)

## Trade-offs and Alternatives

TODO

## Conclusion

TODO
