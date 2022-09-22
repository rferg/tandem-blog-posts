# Rails on Mediator Part 1: The Mediator Pattern and Thin Controllers

Are you having trouble managing the complexity of your Ruby on Rails application?  Are your models tangled together and difficult to change?  Do your controllers contain a lot of complex logic making them difficult to understand and test?  In this 3-part series, I'll suggest some ways that using a mediator to organize parts of your application could help you manage this complexity.  Read on to learn how to make your controllers thinner and decouple them from the rest of your application, how to [use domain events to decouple your models](../part-2-domain-events/article.md), and how to [implement cross-cutting concerns without adding more code to your controllers](../part-3-crosscutting-and-tradeoffs/article.md).

## Preface

TODO

## Mediator Pattern

TODO

## Mediator as "Message Bus"

TODO

```ruby
class MyRequest < Mediate::Request
  attr_reader :message

  def initialize(message)
    @message = message
    super()
  end
end
```

```ruby
class MyRequestHandler < Mediate::RequestHandler
  handles MyRequest

  def handle(request)
    "Received: #{request.message}"
  end
end
```

```ruby
class MyRequest < Mediate::Request
  # same as above...
  handle_with ->(request) { "Received: #{request.message}" }
end
```

```ruby
request = MyRequest.new('hello')
response = Mediate.dispatch(request)
puts response # 'Received: hello'
```

## Thin, Decoupled Controllers

TODO
