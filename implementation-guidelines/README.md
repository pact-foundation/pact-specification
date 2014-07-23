# Consumer and Provider

* Remember to not assume at every stage that the request/response bodies are JSON - this will make implementing XML much easier down the track. The Ruby library uses differs and diff formatters that are configured based on the content type of the request/response.

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

## Mock Service

### Setting up interactions
* The mock service should allow more than one interaction to be expected at the same time, because a client class, or a UI may need to make multiple requests to execute one method.
* More than one mock service should be able to run at a time so that the library can be used for scenarios where multiple backend servers are used (eg. a rich client UI).

### Handling requests
* When a request comes in to the mock service, the request is compared with each interaction that has been registered with the mock service to find the right response to return.
* An interaction is considered a "candidate" for a match if the method and path match.
* If no interaction matches the given request, then a 500 error should be returned by the mock service. The diffs for all candidate interactions should be logged and returned in the response body to assist with debugging.
* If more than one interaction matches the given request, then a 500 error should be returned by the mock service, with a helpful error message. Each matching interaction should be logged and returned in the response body to assist with debugging.
* If exactly one interaction matches the given request, than the corresponding response should be returned, and that interaction should be marked as received.

### Verifying after each test
* After each test, a call should be made to the mock service to verify that all the expected interactions have occured, and that no unexpected interactions have occurred. If either of these is not true, then the test should fail with a helpful error message indicating which interactions were missing completely, which were "incorrect", and which were unexpected. An interaction is considered "Incorrect" if the request method and path match, but the headers, body or query did not match.

### Differentiating between an administrative request and an actual request
* Each language may implement the calls to set up and verify the interactions in different ways, but as a suggestion, the ruby impl uses the HTTP header 'X-Pact-Mock-Service' to identify requests that are "administrative" (eg. setting up an expectation, verifying after a test) - all other requests will be treated as requests coming from the client under test.

# Provider

## Interaction filters

TODO

## Verifying arbitrary pacts

TODO

