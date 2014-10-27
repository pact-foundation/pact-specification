# Consumer and Provider

* Remember to not assume at every stage that the request/response bodies are JSON - this will make implementing XML much easier down the track. The Ruby library uses differs and diff formatters that are configured based on the content type of the request/response.

## Implement "diff", not "match?"

Rather than implementing code the determine whether or not an actual request or response matches an expected request or response, write code that detects the differences between your expected and your actual. You can then use this diff to:

1. Determine if an expected request/response matches an actual request/response by asserting that the diff between two objects is empty.
2. Display any differences in a user friendly format. The output of this diff will be displayed to the user when ever a Pact test fails, in both the consumer project and the provider verification, so it is important to do it well.

## Diff display

The most user friendly format for displaying a JSON diff that I have found after a year of experimenting is [this](https://github.com/realestate-com-au/pact/blob/master/documentation/configuration.md#unix) one.

To create this output, there are two classes involved - the Differ, which calculates the diff between expected and actual, and the DiffFormatter, which takes the diff and makes a nice human readable output of it.

The "diff" is a nested object tree that is basically a copy of the expected tree, but at the paths in the tree where the values do not match, instead of the value, there is a Difference object that contains the expected and the actual values. Eg.

Expected request body:

```json
{
   "person" : {
     "firstname" : "Fred",
     "lastname" : "Smith",
     "favouriteColours" : ["red"],
     "children": ["John","Sue"],
     "address" : {
      "street" : "123 Some St"
     }
   }
}

```

Actual request body:

```json
{
   "person" : {
     "firstname" : "Mary",
     "numberOfLegs" : 2,
     "favouriteColours" : ["red", "green"],
     "children": ["John"]
   }
}

```

Diff:

```ruby
{
   "person" => {
     "firstname" => Difference.new("Fred", "Mary"),
     "lastname" => Difference.new("Smith", KeyNotFound),
     "favouriteColours" => ["red", UnexpectedIndex],
     "children" => ["John", IndexNotFound],
     "numberOfLegs" => Difference.new(UnexpectedKey, 2),
     "address" => Difference.new({"street" => "123 Some St"}, KeyNotFound)
   }
}
```

To create the "unix" style diff string:

1. Recreate the "expected" by walking the diff and collecting the expected values from the Differences.
2. Recreate the "actual" by walking the diff and collecting the actual values from the Differences.
3. Convert both the "expected" and "actual" object trees to JSON.
4. Perform a text diff between the two strings using a third party library.


# Consumer

## DSL

TODO

## 1. Interaction uniqueness
1. Within a pact, the combination of description and provider state should be unique. The interaction list in the pact file should be a Set - if two identical interactions are defined, then only one should be included in the pact file.

2. If an interaction with the same description and provider state, but differing in some other way, is defined, then an error should be thrown indicating that the developer should change either the description or the provider state. The reason this is important is 1. for the sake of meaningful documentation 2. it allows one interaction to be run at a time when verifying a pact by specifying the description and provider state of the interaction, and 3. the Ruby implementation has the option to merge interactions into an existing file when only one test is run (otherwise, all the other interactions get wiped) and there needs to be a unique key to work out which interaction needs to be updated.

## Mock Service

### Setting up interactions
1. The mock service should allow more than one interaction to be expected at the same time, because a client class, or a UI may need to make multiple requests to execute one method.
2. More than one mock service should be able to run at a time so that the library can be used for scenarios where multiple backend servers are used (eg. a rich client UI).

### Handling requests
1. When a request comes in to the mock service, the request is compared with each interaction that has been registered with the mock service to find the right response to return.
1. The rules for determining whether a request "matches" or not are defined by the pact-specification. It must "match" the path, query, headers and body according to the pact specification matching rules.
1. If no interactions match the given request, then a 500 error should be returned by the mock service with an error indicating that no matches have been found. Include a list of the registered interactions to assist with debugging.
1. If more than one interaction matches the given request, then a 500 error should be returned by the mock service, with a helpful error message. Each matching interaction should be logged and returned in the response body to assist with debugging.
1. If exactly one interaction matches the given request, than the corresponding response should be returned, and that interaction should be marked as received.

### Verifying after each test
1. After each test, a call should be made to the mock service to verify that all the expected interactions have occured, and that no unexpected interactions have occurred. If either of these is not true, then the test should fail with a helpful error message indicating which expected interactions were not recieved, which unexpected request were recieved.

### Improving usability of error responses from the mock service
Once the logic described above is implemented, there are are some ways to make the error responses more user friendly.
1. When no matching interaction is found, for each registered request that has a matching method and path, include the diff between it and the incoming request in the error response.
2. In the verify response, for each incoming request that didn't have a matching expectation, include a diff for each interaction with a matching method and path.

### Differentiating between an administrative request and an actual request
1. Each language may implement the calls to set up and verify the interactions in different ways, but as a suggestion, the ruby impl uses the HTTP header 'X-Pact-Mock-Service' to identify requests that are "administrative" (eg. setting up an expectation, verifying after a test) - all other requests will be treated as requests coming from the client under test.

# Provider

## Verification
1. Each interaction should be verified independently without any data leaking from a previous interaction.
1. To this end, a developer should be able to specify a "setup" hook and an "teardown" hook that will run before/after each interaction, as well as a setup/teardown hook for each provider state. 
1. Provider states should be able to be scoped by the consumer name. So, for example "a thing exists" 
 
## Interaction filters

1. The interactions to be verified by the verification task should be able to be filtered from the command line using the environment variables PACT_DESCRIPTION and PACT_PROVIDER_STATE. This will save your sanity when trying to develop the provider, as displaying 20 errors to the screen at once will be overwhelming and unhelpful. For example, `PACT_DESCRIPTION="a request for something" PACT_PROVIDER_STATE="something exists" rake pact:verify`
1.  `PACT_PROVIDER_STATE=""` should match interactions where there is no provider state specified.

## Verifying arbitrary pacts

TODO

