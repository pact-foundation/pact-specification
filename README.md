# Version 4.0 (V4) - Draft

This describes version 4.0 of the Pact file format, as well as associated behaviour for matching and verifying them. For all the changes, refer to the RFC [#71](https://github.com/pact-foundation/pact-specification/issues/71)

## File format

The V4 file is a JSON formatted text file with the following entities:

- consumer
- provider
- interactions
- metadata

### Consumer

This stores details about the consumer of the interaction.

|Field|Type|Description|
|-----|----|-----------|
|name|string|the consumer name|

```json
{
  "consumer": {
    "name": "consumer"
  }
}
```

### Provider

This stores details about the provider of the interaction.

|Field|Type|Description|
|-----|----|-----------|
|name|string|the provider name|

```json
{
  "provider": {
    "name": "provider"
  }
}
```

### Interactions

The interactions capture all the different types of interactions for the contract in an array. Each interaction
must have a `type` attribute to indicate the what type of interaction it is, a `description` and `key`. The key
must be a unique string value for each interaction in the Pact file. For compatibility with previous versions, it can be the 
description, or the description and any provider states. A hash calculated on the contents of the interaction makes
sense to use, but a UUID can also be used.

For example:

```json
{
  "interactions": [
    {
      "type": "Synchronous/HTTP",
      "description": "a retrieve Mallory request",
      "key": "8d1495aa",
      ...
    }
  ]
}
```

|Field|Type|Description|
|-----|----|-----------|
|type|string|The type of the interaction (Synchronous/HTTP, Asynchronous/Messages, Synchronous/Messages, etc.)|
|description|string|A description for the interaction. Must be unique within the Pact file|
|key|string|Unique value for the interaction.|

### Metadata

This stores the metadata associated with the pact file. It is a key/value map, and may contain any data
that can be stored in JSON format. All V4 pact files must have a `pactSpecification` key with a `version`
attribute of `4.0`.

```json
{
    "metadata": {
      "pactSpecification": {
        "version": "4.0"
      }
    }
}
```

It is also preferable for the version of the Pact implementation to be stored in the metadata in a similar format.

For example:

```json
{
  "metadata": {
    "pactSpecification": {
      "version": "4.0"
    },
    "pact-jvm": {
      "version": "4.1.7"
    }
  }
}
```
