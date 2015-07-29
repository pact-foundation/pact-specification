# Introduction

["Pact"](https://github.com/realestate-com-au/pact) is an implementation of "consumer driven contract" testing that allows mocking of responses in the consumer codebase, and verification of the interactions in the provider codebase. The initial implementation was written in Ruby for Rack apps, however a consumer and provider may be implemented in different programming languages, so the "mocking" and the "verifying" steps would be best supported by libraries in their respective project's native languages. Given that the pact file is written in JSON, it should be straightforward to implement a pact library in any language, however, to get the best experience and most reliability of out mixing pact libraries, the matching logic for the requests and responses needs to be identical. There is little confidence to be gained in having your pacts "pass" if the logic used to verify a "pass" is inconsistent between implementations.

To support consistency of matching logic, this specification has been developed as a benchmark that all pact libraries can check themselves against if they want to ensure consistency with other pact libraries.

### Pact Specification Philosophy

* Be as strict as we reasonably can with what we send out (requests). We should know and be able to control exactly what a consumer is sending out. Information should not be allowed to "leak" silently.
* Be as loose as we reasonably can with what we accept (responses). A provider should be able to send extra information that this particular consumer does not care about, without breaking this consumer.
* When writing the matching rules, err on the side of being more strict now, because it will break fewer things to be looser later, than to get stricter later.

Note: One implications of this philosophy is that you cannot verify, using pact, that a key or a header will _not_ be present in a response. You can only verify what _is_.

### Version 3.0 (WIP)

Version 3.0 introduces the following changes from 2.0:

#### Introduces messages for services that communicate via event streams and message queues

This proposal is to introduce a non-request/response interaction to support message queues where the flow may be one way. Pact file format will be expanded to be able
to include something like

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

Currently the query strings are stored as a single string, e.g.:

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

Currently, provider states are defined as a descriptive string. There is no way to infer the data required for the state
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

The current matchers only allow one matcher to be defined for a value. There is no way to combine matchers to do something
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
    "path": [
      {"match": "A"}
    ]
  }
}
```

This will allow expressions like `HEADERY: ValueA, ValueB` with

```json
{
  "matchingRules": {
    "$.header.HEADERY": [
      {"match": "include", "value": "ValueA"},
      {"match": "include", "value": "ValueB"}
    ]
  }
}
```

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
| JVM | ```java new Date().parse("dd-MM-yyyy", "29-02-2001")``` | Thu Mar 01 00:00:00 AEDT 2001 |
| JVM Jodatime | DateTime dateTime  = DateTime.parse("29-02-2001", DateTimeFormat.forPattern("dd-MM-yyyy")) | org.joda.time.IllegalFieldValueException: Cannot parse "29-02-2001": Value 29 for dayOfMonth must be in the range [1,28] |
| Javascript | new Date(Date.parse('Feb 29, 2001')).toString() | Thu Mar 01 2001 00:00:00 GMT+1100 (AEDT) |
| Ruby | Date.strptime('29-02-2001', '%d-%m-%Y') | invalid date (ArgumentError) |


### References

* https://groups.google.com/forum/#!topic/pact-dev/T3TYHJWWw2c
* https://github.com/DiUS/pact-jvm/issues/97
* https://groups.google.com/forum/#!topic/pact-support/d0nXzVi1Nf0
* https://groups.google.com/forum/#!topic/pact-support/YHw7hgD1d4g
