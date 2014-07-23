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