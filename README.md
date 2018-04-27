# Introduction

["Pact"](https://github.com/realestate-com-au/pact) is an implementation of "consumer driven contract" testing that allows mocking of responses in the consumer codebase, and verification of the interactions in the provider codebase. The initial implementation was written in Ruby for Rack apps, however a consumer and provider may be implemented in different programming languages, so the "mocking" and the "verifying" steps would be best supported by libraries in their respective project's native languages. Given that the pact file is written in JSON, it should be straightforward to implement a pact library in any language, however, to get the best experience and most reliability of out mixing pact libraries, the matching logic for the requests and responses needs to be identical. There is little confidence to be gained in having your pacts "pass" if the logic used to verify a "pass" is inconsistent between implementations.

To support consistency of matching logic, this specification has been developed as a benchmark that all pact libraries can check themselves against if they want to ensure consistency with other pact libraries.

### Pact Specificaton Philosophy

* Be as strict as we reasonably can with what we send out (requests). We should know and be able to control exactly what a consumer is sending out. Information should not be allowed to "leak" silently.
* Be as loose as we reasonably can with what we accept (responses). A provider should be able to send extra information that this particular consumer does not care about, without breaking this consumer.
* When writing the matching rules, err on the side of being more strict now, because it will break fewer things to be looser later, than to get stricter later.

Note: One implications of this philosophy is that you cannot verify, using pact, that a key or a header will _not_ be present in a response. You can only verify what _is_.

### Version 1.1.0

#### Request matching

##### Request method

Exact string match, case insensitive.

##### Request path

Exact string match, case sensitive, as paths are generally case sensitive. Trailing slashes should not be ignored, as they could potentially have significance.

##### Request query string

Query strings must have the same key-value pairs. Keys can be present in any order, but when the same key occurs multiple
times, the values must be in the same order.

##### Request headers

Exact string match for expected header names and values. Allow unexpected headers to be sent out, as frameworks and network utilities are likely to set their own headers (eg. User-Agent), and it would increase the maintenance burden to have to track all of those.

##### Request body

* Do not allow unexpected keys to be sent in the body. See "Pact Specificaton Philosophy" in main README.
* Do not allow unexpected items in an array. Most parsing code will do a "for each" on an array, and if we expect one item, but two go out, we have "leaked" information.

#### Response matching

##### Response status

Exact integer match.

##### Response headers

Exact string match for expected header names and values. Allow unexpected headers to be sent back, as in reality, as extra headers will be added by network utilities and server frameworks.

##### Response body

* Allow unexpected keys to be sent back in the body. See "Pact Specificaton Philosophy" in main README.
* Do not allow unexpected items in an array. Most parsing code will do a "for each" on an array, and if we expect one item, but receive two, we might not have exercised the correct code to handle that second item in our consumer tests.

### Differences from V1

The main difference from V1 is the semantics around the body element in the pact files. This specification defines the
following conditions:

#### Body is present

If the body of the request or response is present, then follow the normal rules for matching the bodies.

#### Body is absent

If there is no body in the pact file, then this indicates that the body contents are not important, and can be ignored.

#### Body is present, but empty

If the body is present in the pact file, but is an empty string, then this indicates that the request or response body must be
empty.

#### Body is present, but is null

This is a side effect of JSON and language implementations with NULL values. It has the following semantics:

1. Where the content type is `application\json`, this represents a valid JSON document consisting of the single JSON value
of `null`. It may be treated as either an empty body or follow the rules for matching bodies. The preference would be to
treat it as a JSON body and use an empty string for an absent body.
2. For other content types, it is treated as an empty body for matching purposes.

#### Content type of the body

By default, the content type comes from the `Content-Type` header. The following rules determine the content type:

1. If there is a `Content-Type` header, the value of the header determines the content type.
2. If the `Content-Type` header, is not present, then:
    1. OPTIONAL - The content type can be determined from the first few characters of the body (know as a magic number test).
    2. default to either `application/json` (as V1) or `text/plain`.

## Example

This is an example of a pact file:

```json
{
  "provider": {
    "name": "Alice Service"
  },
  "consumer": {
    "name": "Consumer"
  },
  "interactions": [
    {
      "providerState" : "Good Mallory exists",
      "description": "a retrieve Mallory request",
      "request": {
        "method": "GET",
        "path": "/mallory",
        "query": "name=ron&status=good"
      },
      "response": {
        "status": 200,
        "headers": {
          "Content-Type": "text/html"
        },
        "body": "\"That is some good Mallory.\""
      }
    }
  ],
  "metadata": {
    "pactSpecification": {
      "version": "1.1.0"
    },
    "pact-jvm": {
      "version": "1.0.0"
    }
  }
}
```
