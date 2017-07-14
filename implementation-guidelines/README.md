# Implementation guidelines

Our current approach for implementing pact in a new language is to create a native DSL that calls the a [standalone package][pact-ruby-standalone] of the Ruby implementation via HTTP and command line. This approach is temporary, as we're working on a Rust implementation that will be able to provide FFI bindings and integrate much more elegantly.

You can see which Pact implementations use the standalone packages in the [Pact Dependency Graph of Doom][dependency-graph].

## Consumer

You will need to implement a Pact DSL in the idioms of your language that allows one or more mock services to be started up on specific ports. Note that you need to use a separate mock service instance for each provider. Run `pact-mock-service help start` for all the options that you will need to expose. Ideally, your test framework will take care of the mock service process management so your users don't have to do anything.

```ruby
Pact.service_consumer "Zoo App" do
  has_pact_with "Animal Service" do
    mock_service :animal_service do
      port 1234
      pact_specification_version '2'
    end
  end
end

```

You will need to implement a DSL using the idioms of your langauge that allows definition of one _or more_ interactions to be set up on a mock server.

```ruby

   animal_service.given("an alligator exists").
        upon_receiving("a request for an alligator").
        with(method: 
          :get, 
          path: '/alligator', 
          query: {foo: ["bar", "wiffle"]}).
        will_respond_with(
          status: 200,
          headers: {'Content-Type' => 'application/json'},
          body: {name: 'Betty'} ).
        given(...).
        upon_receiving(...)
```

You will need to implement a mock service client that makes HTTP calls to the mock service. See this [gist](https://gist.github.com/bethesque/9d81f21d6f77650811f4) for examples of the HTTP calls that need to be made.

*Notes on the pact format between the client and the mock service*: To communicate between the DSL and the mock service, the Ruby code still uses the original Pact format that uses Ruby specific JSON ("json_class": "Pact::Term"...) to serialise the terms, regexps and SomethingLikes. You can use the newer pact-specification v2 matching code if you like as both are accepted, but TBH the old Ruby way is actually easier to implement!

[pact-ruby-standalone]: https://github.com/pact-foundation/pact-ruby-standalone
[dependency-graph]: https://github.com/pact-foundation/README/blob/master/dependency_graph.md
