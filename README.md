# Introduction

["Pact"](https://github.com/realestate-com-au/pact) is an implementation of "consumer driven contract" testing that
allows mocking of responses in the consumer codebase, and verification of the interactions in the provider codebase.
The initial implementation was written in Ruby for Rack apps, however a consumer and provider may be implemented in
different programming languages, so the "mocking" and the "verifying" steps would be best supported by libraries in
their respective project's native languages. Given that the pact file is written in JSON, it should be straightforward
to implement a pact library in any language, however, to get the best experience and most reliability of out mixing
pact libraries, the matching logic for the requests and responses needs to be identical. There is little confidence to
be gained in having your pacts "pass" if the logic used to verify a "pass" is inconsistent between implementations.

To support consistency of matching logic, this specification has been developed as a benchmark that all pact libraries
can check themselves against if they want to ensure consistency with other pact libraries.

### Pact Specification Philosophy

* Be as strict as we reasonably can with what we send out (requests). We should know and be able to control exactly
what a consumer is sending out. Information should not be allowed to "leak" silently.
* Be as loose as we reasonably can with what we accept (responses). A provider should be able to send extra information
that this particular consumer does not care about, without breaking this consumer.
* When writing the matching rules, err on the side of being more strict now, because it will break fewer things to be
looser later, than to get stricter later.

Note: One implications of this philosophy is that you cannot verify, using pact, that a key or a header will _not_ be
present in a response. You can only verify what _is_.

### Version 2.0

See this [gist](https://gist.github.com/bethesque/5a35a3c1cb9fdab6dce7) for an idea of where version 2.0 is came from.

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

* Allow unexpected keys to be sent back in the body. See "Pact Specification Philosophy" in main README.
* Do not allow unexpected items in an array. Most parsing code will do a "for each" on an array, and if we expect one item, but receive two, we might not have exercised the correct code to handle that second item in our consumer tests.

### Differences from V1

The main difference from V1 is the semantics around the body element in the pact files and the addition of matchers. This specification defines the
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

### Matchers

#### Matching Rules

Pact supports extending the matching rules on each type of object (Request or Response) with a `matchingRules` element in the pact file.
This is a map of JSON path strings to a matcher. When an item is being compared, if there is an entry in the matching
rules that corresponds to the path to the item, the comparison will be delegated to the defined matcher. Note that the
matching rules cascade, so a rule can be specified on a value and will apply to all children of that value.

#### Matcher Path expressions

Pact does not support the full JSON path expressions, only ones that match the following rules:

1. All paths start with a dollar (`$`), representing the root.
2. All path elements are separated by periods (`.`), except array indices which use square brackets (`[]`).
3. The second element of the path is the http type that the matcher is applied to (e.g., `$.body` or `$.header`).
4. Path elements represent keys.
5. A star (`*`) can be used to match all keys of a map or all items of an array (one level only).

So the expression `$.body.item1.level[2].id` will match the highlighted item in the following body:

```js
{
  "item1": {
    "level": [
      {
        "id": 100
      },
      {
        "id": 101
      },
      {
        "id": 102 // <---- $.body.item1.level[2].id
      },
      {
        "id": 103
      }
    ]
  }
}
```

while `$.body.*.level[*].id` will match all the ids of all the levels for all items.

##### Matcher selection algorithm

Due to the star notation, there can be multiple matcher paths defined that correspond to an item. The first, most
specific expression is selected by assigning weightings to each path element and taking the product of the weightings.
The matcher with the path with the largest weighting is used.

* The root node (`$`) is assigned the value 2.
* Any path element that does not match is assigned the value 0.
* Any property name that matches a path element is assigned the value 2.
* Any array index that matches a path element is assigned the value 2.
* Any star (`*`) that matches a property or array index is assigned the value 1.
* Everything else is assigned the value 0.

So for the body with highlighted item:

```js
{
  "item1": {
    "level": [
      {
        "id": 100
      },
      {
        "id": 101
      },
      {
        "id": 102 // <--- Item under consideration
      },
      {
        "id": 103
      }
    ]
  }
}
```

The expressions will have the following weightings:

| expression | weighting calculation | weighting |
|------------|-----------------------|-----------|
| $ | $(2) | 2 |
| $.body | $(2).body(2) | 4 |
| $.body.item1 | $(2).body(2).item1(2) | 8 |
| $.body.item2 | $(2).body(2).item2(0) | 0 |
| $.header.item1 | $(2).header(0).item1(2) | 0 |
| $.body.item1.level | $(2).body(2).item1(2).level(2) | 16 |
| $.body.item1.level[1] | $(2).body(2).item1(2).level(2)[1(2)] | 32 |
| $.body.item1.level[1].id | $(2).body(2).item1(2).level(2)[1(2)].id(2) | 64 |
| $.body.item1.level[1].name | $(2).body(2).item1(2).level(2)[1(2)].name(0) | 0 |
| $.body.item1.level[2] | $(2).body(2).item1(2).level(2)[2(0)] | 0 |
| $.body.item1.level[2].id | $(2).body(2).item1(2).level(2)[2(0)].id(2) | 0 |
| $.body.item1.level[*].id | $(2).body(2).item1(2).level(2)[*(1)].id(2) | 32 |
| $.body.\*.level[\*].id | $(2).body(2).*(1).level(2)[*(1)].id(2) | 8 |

So for the item with id 102, the matcher with path `$.body.item1.level[1].id` and weighting 64 will be selected.

#### Supported matchers

The following matchers are supported:

| matcher | example configuration | description |
|---------|-----------------------|-------------|
| Regex | `{ "match": "regex", "regex": "\\d+" }` | This executes a regular expression match against the string representation of a values. |
| Type | `{ "match": "type" }` | This executes a type based match against the values, that is, they are equal if they are the same type. |
| MinType | `{ "match": "type", "min": 2 }` | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the minimum. |
| MaxType | `{ "match": "type", "max": 10 }` | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the maximum. |

If no matcher is specified an equality matcher will be used, which relies on the equals operator.

## Example

This is an example of a pact file:

```json
{
    "provider": {
        "name": "266_provider"
    },
    "consumer": {
        "name": "test_consumer"
    },
    "interactions": [
        {
            "description": "get all users for max",
            "request": {
                "method": "GET",
                "path": "/idm/user"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json; charset=UTF-8"
                },
                "body": [
                    [
                        {
                            "email": "rddtGwwWMEhnkAPEmsyE",
                            "id": "eb0f8c17-c06a-479e-9204-14f7c95b63a6",
                            "userName": "AJQrokEGPAVdOHprQpKP"
                        }
                    ]
                ],
                "matchingRules": {
                    "$.body[0][*].email": {
                        "match": "type"
                    },
                    "$.body[0][*].id": {
                        "regex": "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
                    },
                    "$.body[0]": {
                        "max": 5,
                        "match": "type"
                    },
                    "$.body[0][*].userName": {
                        "match": "type"
                    }
                }
            },
            "providerState": "a user with an id named 'user' exists"
        },
        {
            "description": "get all users for min",
            "request": {
                "method": "GET",
                "path": "/idm/user"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json; charset=UTF-8"
                },
                "body": [
                    [
                        {
                            "email": "DPvAfkCZpOBZWzKYiDMC",
                            "id": "95d0371b-bf30-4943-90a8-8bb1967c4cb2",
                            "userName": "GIUlVKoiLdHLYNKGbcSy"
                        }
                    ]
                ],
                "matchingRules": {
                    "$.body[0][*].email": {
                        "match": "type"
                    },
                    "$.body[0][*].id": {
                        "regex": "[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}"
                    },
                    "$.body[0]": {
                        "min": 5,
                        "match": "type"
                    },
                    "$.body[0][*].userName": {
                        "match": "type"
                    }
                }
            },
            "providerState": "a user with an id named 'user' exists"
        }
    ],
    "metadata": {
        "pact-specification": {
            "version": "2.0.0"
        },
        "pact-jvm": {
            "version": "3.2.11"
        }
    }
}
```
