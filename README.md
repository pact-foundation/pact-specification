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

### Version 4.0 (WIP)

Version 4.0 is a work in progress and introduces the following changes from 3.0:

#### Allow schemas to be defined request, response and message content

This would add a schema element to each body/content. The schema would be used to validate the request bodies in the
consumer tests and the response bodies in the provider bodies. JSONSchema could be used for JSON bodies, and for XML XSD or
something like schematron.

#### Matching times and dates in a cross-platform manner

Regular expression matching only allows syntactic evaluation of times and dates. It is quite hard to define expressions
that could determine that dates like '29-02-2001' are invalid. Dates also have different defaulting behavior on different platforms.

e.g:

| Platform | Expression | Result |
| -------- | ---------- | ------ |
| JVM | `new Date().parse("dd-MM-yyyy", "29-02-2001")` | Thu Mar 01 00:00:00 AEDT 2001 |
| JVM Jodatime | `DateTime dateTime  = DateTime.parse("29-02-2001", DateTimeFormat.forPattern("dd-MM-yyyy"))` | org.joda.time.IllegalFieldValueException: Cannot parse "29-02-2001": Value 29 for dayOfMonth must be in the range [1,28] |
| Javascript | `new Date(Date.parse('Feb 29, 2001')).toString()` | Thu Mar 01 2001 00:00:00 GMT+1100 (AEDT) |
| Ruby | `Date.strptime('29-02-2001', '%d-%m-%Y')` | invalid date (ArgumentError) |

#### Additional Matchers

The following additional matchers are proposed:

| Matcher | Description | Example JSON |
| ------- | ----------- | ------------ |
| ignored | Indicates an optional attribute that should be ignored | `{ "match": "ignore" }` |
| nullValue | Matches NULL values only. This is used to provide matcher defintions like `or(decimal, nullValue)` | `{ "match": "nullValue" }` |
| absent | Indicates an attribute that must never be present | `{ "match": "absent" }` |

### Ignoring the keys in a map

Some payloads may contain a map of IDs to values. In these cases, the keys should be ignored and only the values compared. This can be achieved by defining a new `mapValues` matcher on the map. For reference, see [#47](https://github.com/pact-foundation/pact-specification/issues/47) and [Pact-JVM #313](https://github.com/DiUS/pact-jvm/issues/313).

### References

* https://groups.google.com/forum/#!topic/pact-support/YHw7hgD1d4g

### Semantics around body values

The semantics around the body element in the pact files is defined by the following rules:

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

#### Matcher Path expressions in bodies

Pact does not support the full JSON path expressions, only ones that match the following rules:

1. All paths start with a dollar (`$`), representing the root.
2. All path elements are separated by periods (`.`), except array indices which use square brackets (`[]`).
3. The second element of the path is the http type that the matcher is applied to (e.g., `$.body` or `$.header`).
4. Path elements represent keys.
5. A star (`*`) can be used to match all keys of a map or all items of an array (one level only).

So the expression `$.item1.level[2].id` will match the highlighted item in the following body:

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
        "id": 102 // <---- $.item1.level[2].id
      },
      {
        "id": 103
      }
    ]
  }
}
```

while `$.*.level[*].id` will match all the ids of all the levels for all items.

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
| $.item1 | $(2).item1(2) | 4 |
| $.item2 | $(2).item2(0) | 0 |
| $.item1.level | $(2).item1(2).level(2) | 8 |
| $.item1.level[1] | $(2).item1(2).level(2)[1(2)] | 16 |
| $.item1.level[1].id | $(2).item1(2).level(2)[1(2)].id(2) | 32 |
| $.item1.level[1].name | $(2).item1(2).level(2)[1(2)].name(0) | 0 |
| $.item1.level[2] | $(2).item1(2).level(2)[2(0)] | 0 |
| $.item1.level[2].id | $(2).item1(2).level(2)[2(0)].id(2) | 0 |
| $.item1.level[*].id | $(2).item1(2).level(2)[*(1)].id(2) | 16 |
| $.\*.level[\*].id | $(2).*(1).level(2)[*(1)].id(2) | 8 |

So for the item with id 102, the matcher with path `$.item1.level[1].id` and weighting 32 will be selected.

#### Supported matchers

The following matchers are supported:

| matcher | example configuration | description |
|---------|-----------------------|-------------|
| Equality | `{ "match": "equality" }` | This is the default matcher, and relies on the equals operator |
| Regex | `{ "match": "regex", "regex": "\\d+" }` | This executes a regular expression match against the string representation of a values. |
| Type | `{ "match": "type" }` | This executes a type based match against the values, that is, they are equal if they are the same type. |
| MinType | `{ "match": "type", "min": 2 }` | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the minimum. |
| MaxType | `{ "match": "type", "max": 10 }` | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the maximum. |
| Integer | `{ "match": "integer" }` | Matches the example by type, and ensures that the actual value is also an integer  |
| Decimal | `{ "match": "decimal" }` | Matches the example by type, and ensures that the actual value is also a decimal number (has decimal places) |

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
