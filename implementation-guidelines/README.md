# Consumer and Provider

* Remember to not assume at every stage that the request/response bodies are JSON - this will make implementing XML much easier down the track. The Ruby library uses differs and diff formatters that are configured based on the content type of the request/response.

## Diff display

The most user friendly format for displaying a JSON diff that I have found after a year of experimenting is [this](https://github.com/realestate-com-au/pact/blob/master/documentation/configuration.md#unix) one.

To create this diff:

1. Take the expected and actual object trees  
2. If this is a response body, remove any unexpected keys from the actual, as unexpected keys are allowed  
3. Order the keys the same in both expected and actual  
4. Convert both to JSON strings  
5. Perform a text diff using a third party diff library (prefereably one that uses colour!)  

Ok, the above is actually a lie, but it will help you understand what actually happens.

The json differ actually returns a Diff object, which is a nested object tree that is basically a copy of the expected tree, but at the paths in the tree where the values do not match, instead of the value, there is a Difference object that contains the expected and the actual values. Eg.

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

From this diff, the expected and actual object trees are recreated, and THEN they are turned into JSON, and a text diff is performed on those two strings. The reason it is done this way is that what is considered a match is NOT the same as what a straight text diff considers a match.


# Consumer

## DSL

TODO

## Mock Service

* The mock service should allow more than one interaction to be expected at the same time, because a client class, or a UI may need to make multiple requests to execute one method.
* When a request comes in to the mock service, the request is compared with each interaction that has been registered with the mock service to find the right response to return.
* An interaction is considered a "candidate" for a match if the method and path match.
* If no interaction matches the given request, then a 500 error should be returned by the mock service. The diffs for all candidate interactions should be logged and returned in the response body to assist with debugging.
* If more than one interaction matches the given request, then a 500 error should be returned by the mock service, with a helpful error message. Each matching interaction should be logged and returned in the response body to assist with debugging.
* If exactly one interaction matches the given request, than the corresponding response should be returned, and that interaction should be marked as received.
* After each test, a call should be made to the mock service to verify that all the expected interactions have occured, and that no unexpected interactions have occurred. If either of these is not true, then the test should fail with a helpful error message indicating which interactions were missing completely, which were "incorrect", and which were unexpected. An interaction is considered "Incorrect" if the request method and path match, but the headers, body or query did not match.

# Provider
