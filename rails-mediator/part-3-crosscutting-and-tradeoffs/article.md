# Rails on Mediator Part 3: Cross-cutting Concerns and Tradeoffs

TODO: INTRO

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
