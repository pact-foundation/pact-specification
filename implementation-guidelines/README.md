# Implementation guidelines

Our current approach for implementing pact in a new language is to create a native DSL that calls the a [standalone package][pact-ruby-standalone] of the Ruby implementation via HTTP and command line. This approach is temporary, as we're working on a [Rust implementation][rust-pact-reference] that will be able to provide FFI bindings and integrate much more elegantly.

You can see which Pact implementations use the standalone packages in the [Pact Dependency Graph of Doom][dependency-graph].

## Consumer

You will need to implement a Pact DSL in the idioms of your language that allows one or more **mock services to be started** up on specific ports. Note that you need to use a separate mock service instance for each provider. Run `pact-mock-service help start` for all the options that you will need to expose. Ideally, your test framework will take care of the mock service process management so your users don't have to do anything.

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

You will need to implement a **DSL** using the idioms of your langauge that allows definition of one _or more_ interactions to be set up on a mock server.

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

You will need to implement a **mock service client** that makes HTTP calls to the mock service. See this [gist](https://gist.github.com/bethesque/9d81f21d6f77650811f4) for examples of the HTTP calls that need to be made.

*Notes on the pact format between the client and the mock service*: To communicate between the DSL and the mock service, the Ruby code still uses the original Pact format that uses Ruby specific JSON ("json_class": "Pact::Term"...) to serialise the Terms, regexps and SomethingLikes. You can use the newer pact-specification v2 matching code if you like as both are accepted, but TBH the old Ruby way is actually easier to implement!

You will need to implement code to **publish pacts to the pact broker**. See the Ruby [Pact Broker Client][pact-broker-client] as an example of the configuration options and functionality you should provide. You can see the pact between the mock service and the client [here][pact-broker-pact-docs] (you will only need to implement `A request to publish a pact` and `A request to tag the production version of Condor` interactions.)
You can read more about publishing and tagging on the Pact Broker [wiki][pact-broker-wiki].
Please note that you should tag the version before publishing the pact to avoid the potential of a pact version being returned for a given `latest` URL when it shouldn't be.

## Provider

You will need to implement a verification task that calls the `pact-provider-verifier`. It will expose configuration code for the URLs of the pacts that need to be verified (local and HTTP endpoints) and expose configuration options for broker authentication. Run `pact-provider-verifier help verify` to see all configuration options you will need to provide.

```ruby
Pact.service_provider "Animal Service" do

  honours_pact_with 'Zoo App' do
    pact_uri '../zoo-app/spec/pacts/zoo_app-animal_service.json'
  end
end
```

Publishing of the verification results will be done automatically by the `pact-provider-verifier` if you set `publish-verification-results` to true and provide a `provider version number`.

For bonus points, you can implement code that retrieves all the pacts for a given provider dynamically from the pact_broker.

[pact-ruby-standalone]: https://github.com/pact-foundation/pact-ruby-standalone
[dependency-graph]: https://github.com/pact-foundation/README/blob/master/dependency_graph.md
[pact-broker-client]: https://github.com/pact-foundation/pact_broker-client
[pact-broker-pact-docs]: https://github.com/pact-foundation/pact_broker-client/blob/master/doc/markdown/Pact%20Broker%20Client%20-%20Pact%20Broker.md
[pact-broker-wiki]: https://github.com/pact-foundation/pact_broker/wiki
[rust-pact-reference]: https://github.com/pact-foundation/pact-reference
