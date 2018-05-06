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

### Version 3.0

Version 3.0 introduces the following changes from 2.0:

#### Introduces messages for services that communicate via event streams and message queues

This proposal is to introduce a non-request/response interaction to support message queues where the flow may be one way.
Pact file format will be expanded to be able to include:

```json
{
    "consumer": {
        "name": "Consumer"
    },
    "provider": {
        "name": "Provider"
    },
    "messages": [
        {
            "description": "Published credit data",
            "providerState": "or maybe 'scenario'? not sure about this",
            "contents": {
                "foo": "bar"
            },
            "metaData": {
              "contentType": "application/json"
            }
        }
    ]
}
```

As some message formats are binary, it may be necessary to Base64 encode the message contents.

#### Query strings are stored as Map instead of strings

In previous versions, the query strings are stored as a single string, e.g.:

```json
"request": {
  "method": "get",
  "path": "/autoComplete/address",
  "query": "max_results=100&state=NSW&term=80+CLARENCE+ST,+SYDNEY+NSW+2000",
}
```

This proposal is to store them in un-encoded map of keys to list of values format:

```json
"request": {
  "method": "get",
  "path": "/autoComplete/address",
  "query": {
    "max_results": ["100"],
    "state": ["NSW"],
    "term": ["80 CLARENCE ST, SYDNEY NSW 2000"]
  }
}
```

#### Allow multiple provider states with parameters

In previous versions, provider states are defined as a descriptive string. There is no way to infer the data required for the state
without encoding the values into the description.

```json
{
  "providerState": "an alligator with the given name Mary exists and the user Fred is logged in"
}
```

The change would be:

```json
{
  "providerStates": [
    {
      "name": "an alligator with the given name exists",
      "params": {"name" : "Mary"}
    }, {
      "name": "the user is logged in",
      "params" : { "username" : "Fred"}
    }
  ]
}
```

#### Allow arrays of matchers to be defined against a matcher path

The V2 matchers only allow one matcher to be defined for a value. There is no way to combine matchers to do something
like value must match A and must match B.

Proposal is  to change

```json
{
  "matchingRules": {
    "path": {"match": "A"}
  }
}
```

to

```json
{
  "matchingRules": {
    "path": {
        "matchers": [
          {"match": "A"}
        ]
    }
  }
}
```

This will allow expressions like `HEADERY: ValueA, ValueB` with

```json
{
  "matchingRules": {
    "header": {
      "HEADERY": {
          "matchers": [
            {"match": "include", "value": "ValueA"},
            {"match": "include", "value": "ValueB"}
          ]
      }
    }
  }
}
```

It would also be required to define whether the matchers should be combined with logical AND (all matchers must match)
or OR (at least one matcher must match). AND should be the default, but there are cases where an OR makes sense.

```json
{
  "matchingRules": {
    "header": {
      "HEADERY": {
          "combine": "AND",
          "matchers": [
            {"match": "include", "value": "ValueA"},
            {"match": "include", "value": "ValueB"}
          ]
      }
    }
  }
}
```

#### Drop the Jsonpath notation for matchers

The matchers are defined in terms of a Jsonpath expression. This has caused some confusion, and is unnecessary. Secondly,
only a subset of Jsonpath is supported. This proposal is drop Jsonpath expressions in the matcher keys, and have request/response
type definitions instead. Jsonpath will still be used internally for matching body elements.

Request Example:

```json
"matchingRules": {
  "path": {
    "matchers": [
        { "match": "regex", "regex": "\\w+" }
      ]
  },
  "query": {
    "Q1": {
        "matchers": [
          { "match": "regex", "regex": "\\w+" }
        ]
    }
  },
  "header": {
    "Accept": {
        "matchers": [
            { "match" : "regex", "regex" : "\\w+" }
        ]
    }
  },
  "body": {
    "$.animals": {
        "matchers": [{"min": 1, "match": "type"}]
    },
    "$.animals[*].*": {
        "matchers": [{"match": "type"}]
    },
    "$.animals[*].children": {
        "matchers": [{"min": 1}]
    },
    "$.animals[*].children[*].*": {
        "matchers": [{"match": "type"}]
    }
  }
}
```

#### Add an equality matcher

Matchers on body elements cascade. Once a matcher is set, there needs to be a way to reset the matching on a child element.
This proposal introduces an equality matcher to reset the matching back to the default.

Example:

```json
"matchingRules": {
  "body": {
    "$.animals": { "matchers": [{"min": 1, "match": "type"}] },
    "$.animals[*].*": { "matchers": [{"match": "type"}] },
    "$.animals[*].children": { "matchers": [{"min": 1}] },
    "$.animals[*].children[*].*": { "matchers": [{"match": "type"}] },
    "$.animals[*].children[*].*.name": { "matchers": [{"match": "equality"}] }
  }
}
```

#### Add an include matcher

Simple matcher to determine if a string is present in a value.

Example:

```json
"matchingRules": {
  "body": {
    "$.value": { "matchers": [{"match": "include", "value": "ValueA"}] },
  }
}
```

#### Add a minmax type matcher

Sometimes it is required to ensure that a collection has both a minimum and maximum size.

Example:

```json
"matchingRules": {
  "body": {
    "$.values": { "matchers": [{"match": "type", "min": "1", "max": "1"}] },
  }
}
```

#### Add more specific type matchers

Type matchers sometimes need to be more specific. Sometimes just matching the type of the example is not enough. Dates
and times are normally encoded as strings (these are addressed in proposal
[Matching times and dates in a cross-platform manner](https://github.com/pact-foundation/pact-specification/tree/version-4#matching-times-and-dates-in-a-cross-platform-manner)). Sometimes
numeric values need to be ensured that they match the specific numeric type. This is especially important for financial
systems, where a rounding error can be catastrophic. The general type matcher will match 100 and 100.01.

The following matchers have been added:

| Matcher | Description | Example |
|---------|-------------|---------|
| Integer | Matches the example by type, and ensures that the actual value is also an integer | `{ "match": "integer" }` |
| Decimal | Matches the example by type, and ensures that the actual value is also a decimal number (has decimal places) | `{ "match": "decimal" }` |
| Null | Matches only null values | `{ "match": "null" }` |

#### Introduce example generators

The example requests and response bodies stored in a pact file are static. The idea being that the pact file represents a
contract that can always be fulfilled if the provider is in the correct state. However, this assumption is not always
correct. In some cases, dates and times may need to be relative to the current date and time, and some things like tokens may
have a very short life span.

An example of the date issue is a provider which only accepts a date value in the current financial year. As soon as we
switch over to a new financial year (or any time period), that pact file can no longer be used.

This proposal introduces the concept of an example value generator, which can replace an example value based on a path
with a dynamically generated one.

```json
{
  "body": {
    "id": 100,
    "description": "Small pack of bolts",
    "processDate": "2015-07-01"
  },
  "generators": {
      "body": {
          "$.id": {
              "type": "RandomDecimal",
              "digits": 5
          },
          "$.description": {
              "type": "RandomString",
              "size": 20
          },
          "$.processDate": {
              "type": "Date"
          }
      }
  }
}
```

Currently supported generators:

| Generator | Attributes | Description | Example JSON |
| --------- | ---------- | ----------- | ------------ |
| RandomInt | min, max   | Generates a random integer value between `min` and `max` values | `{ "type": "RandomInt", "min": 0,  "max": 2147483647 }`
| RandomDecimal | digits  | Generates a random decimal value (BigDecimal) with the provided number of digits | `{ "type": "RandomDecimal", "digits": 6 }`
| RandomHexadecimal | digits  | Generates a random hexadecimal value (String) with the provided number of digits | `{ "type": "RandomHexadecimal", "digits": 8 }`
| RandomString | size  | Generates a random string value of the provided size characters | `{ "type": "RandomString", "size": 20 }`
| Regex | regex  | Generates a random string value from the provided regular expression | `{ "type": "Regex", "regex": "\\d{1,8}" }`
| Uuid | | Generates a random UUID value | `{ "type": "Uuid" }`
| Date | format (Optional) | Generates a Date value from the current date either in ISO format or using the provided format string | `{ "type": "Date", "format": "MM/dd/yyyy" }`
| Time | format (Optional) | Generates a Time value from the current time either in ISO format or using the provided format string | `{ "type": "Time", "format": "HH:mm" }`
| DateTime | format (Optional) | Generates a Date and Time (timestamp) value from the current date and time either in ISO format or using the provided format string | `{ "type": "DateTime", "format": "yyyy/MM/dd - HH:mm:ss.S" }`
| Boolean | | Generates a random boolean value | `{ "type": "RandomBoolean" }`

#### Content-Type header matching should include parameters in the matching

A lot of failures with content types arise when the actual header includes a charset parameter, while the expectation does not.
It is desirable that this not fail matching when the charset is supplied, following Postel's law.

For example, the expected header is set to `application/json` while the actual one is `application/json;charset=UTF-8`.

Here, charset parameter is additional data, so these two values should be equivalent.

### References

* https://groups.google.com/forum/#!topic/pact-dev/T3TYHJWWw2c
* https://github.com/DiUS/pact-jvm/issues/97
* https://groups.google.com/forum/#!topic/pact-support/d0nXzVi1Nf0

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
2. All path elements are either separated by periods (`.`) or use the JSON path bracket notation (square brackets and single quotes around the values: e.g. `['x.y']`), except array indices which use square brackets (`[]`). For elements where the value contains white space or non-alphanumeric characters, the JSON path bracket notation (`['']`) should be used.
3. The second element of the path is the http type that the matcher is applied to (e.g., `$.body` or `$.header`).
4. Path elements represent keys.
5. A star (`*`) can be used to match all keys of a map or all items of an array (one level only).

So the expression `$.item1.level[1].id` will match the highlighted item in the following body:

```js
{
  "item1": {
    "level": [
      {
        "id": 100
      },
      {
        "id": 101 // <---- $.item1.level[1].id
      },
      {
        "id": 102
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
        "id": 101 // <--- Item under consideration
      },
      {
        "id": 102
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
| $.\*.level[\*].id | $(2).*(1).level(2)[*(1)].id(2) | 16 |

So for the item with id 101, the matcher with path `$.item1.level[1].id` and weighting 32 will be selected.

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
        "name": "test_provider_array"
    },
    "consumer": {
        "name": "test_consumer_array"
    },
    "interactions": [
        {
            "description": "java test interaction with a DSL array body",
            "request": {
                "method": "GET",
                "path": "/"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json; charset=UTF-8"
                },
                "body": [
                    {
                        "dob": "07/19/2016",
                        "id": 8958464620,
                        "name": "Rogger the Dogger",
                        "timestamp": "2016-07-19T12:14:39"
                    },
                    {
                        "dob": "07/19/2016",
                        "id": 4143398442,
                        "name": "Cat in the Hat",
                        "timestamp": "2016-07-19T12:14:39"
                    }
                ],
                "matchingRules": {
                    "body": {
                        "$[0].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[1].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        }
                    }
                }
            }
        },
        {
            "description": "test interaction with a array body with templates",
            "request": {
                "method": "GET",
                "path": "/"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json; charset=UTF-8"
                },
                "body": [
                    {
                        "dob": "2016-07-19",
                        "id": 1943791933,
                        "name": "ZSAICmTmiwgFFInuEuiK"
                    },
                    {
                        "dob": "2016-07-19",
                        "id": 1943791933,
                        "name": "ZSAICmTmiwgFFInuEuiK"
                    },
                    {
                        "dob": "2016-07-19",
                        "id": 1943791933,
                        "name": "ZSAICmTmiwgFFInuEuiK"
                    }
                ],
                "matchingRules": {
                    "body": {
                        "$[2].name": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[0].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[1].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[2].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[1].name": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[0].name": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$[0].dob": {
                            "matchers": [
                              { "date": "yyyy-MM-dd" }
                            ]
                        }
                    }
                }
            }
        },
        {
            "description": "test interaction with an array like matcher",
            "request": {
                "method": "GET",
                "path": "/"
            },
            "response": {
                "status": 200,
                "headers": {
                    "Content-Type": "application/json; charset=UTF-8"
                },
                "body": {
                    "data": {
                        "array1": [
                            {
                                "dob": "2016-07-19",
                                "id": 1600309982,
                                "name": "FVsWAGZTFGPLhWjLuBOd"
                            }
                        ],
                        "array2": [
                            {
                                "address": "127.0.0.1",
                                "name": "jvxrzduZnwwxpFYrQnpd"
                            }
                        ],
                        "array3": [
                            [
                                {
                                    "itemCount": 652571349
                                }
                            ]
                        ]
                    },
                    "id": 7183997828
                },
                "matchingRules": {
                    "body": {
                        "$.data.array3[0]": {
                            "matchers": [
                              { "max": 5, "match": "type" }
                            ]
                        },
                        "$.data.array1": {
                            "matchers": [
                              { "min": 0, "match": "type" }
                            ]
                        },
                        "$.data.array2": {
                            "matchers": [
                              { "min": 1, "match": "type" }
                            ]
                        },
                        "$.id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$.data.array3[0][*].itemCount": {
                            "matchers": [
                              { "match": "integer" }
                            ]
                        },
                        "$.data.array2[*].name": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$.data.array2[*].address": {
                            "matchers": [
                              { "regex": "(\\d{1,3}\\.)+\\d{1,3}" }
                            ]
                        },
                        "$.data.array1[*].name": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        },
                        "$.data.array1[*].id": {
                            "matchers": [
                              { "match": "type" }
                            ]
                        }
                    }
                }
            }
        }
    ],
    "metadata": {
        "pactSpecification": {
            "version": "3.0.0"
        },
        "pact-jvm": {
            "version": "3.2.11"
        }
    }
}
```
