# Version 4.0 (V4)

This describes version 4.0 of the Pact file format, as well as associated behaviour for matching and verifying them. 
For all the changes, refer to the RFC [#71](https://github.com/pact-foundation/pact-specification/issues/71)

## JSON Schema

For a declarative, structured format of this version of the specification, see its [JSON Schema](https://raw.githubusercontent.com/pactflow/pact-schemas/main/dist/pact-schema-v4.json).

## File format

The V4 file is a JSON formatted text file with the following entities:

- consumer
- provider
- interactions
- metadata

NOTE: When reading and writing Pact files, we follow [the robustness principle](https://en.wikipedia.org/wiki/Robustness_principle),
so attributes that don't conform to the specification should be ignored and not stop the file from being processed. Ideally, 
a warning should be displayed to indicate that this behaviour has occurred. 

### Consumer

This stores details about the consumer of the interaction.

| Field | Type   | Description       |
|-------|--------|-------------------|
| name  | string | the consumer name |

```json
{
  "consumer": {
    "name": "consumer"
  }
}
```

### Provider

This stores details about the provider of the interaction.

| Field | Type   | Description       |
|-------|--------|-------------------|
| name  | string | the provider name |

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

| Field               | Required? | Type                           | Description                                                                                          |
|---------------------|-----------|--------------------------------|------------------------------------------------------------------------------------------------------|
| type                | required  | string                         | The type of the interaction (Synchronous/HTTP, Asynchronous/Messages, Synchronous/Messages, etc.)    |
| description         | required  | string                         | A description for the interaction. Must be unique within the Pact file                               |
| key                 | optional  | string                         | Unique value for the interaction. Can be auto-generated if not specified.                            |
| providerStates      | optional  | List[ProviderState]            | Provider states required to verify the interaction                                                   |
| pending             | optional  | boolean                        | Mark this interaction as pending. See https://docs.pact.io/pact_broker/advanced_topics/pending_pacts |
| comments            | optional  | Map[string, JSON]              | Comments that are applied to the interaction. See comments section below                             |
| pluginConfiguration | optional  | Map[string, Map[string, JSON]] | Configuration applied by a plugin. See Pact plugins for more details                                 |
| interactionMarkup   | optional  | InteractionMarkup              | Markup to use to render the interaction in a UI. This will be supplied when using plugins.           |

#### Synchronous/HTTP

This is the original Pact interaction (V1/V2), and represents a synchronous request/response, mainly executed over
HTTP. It contains a request entity with all the HTTP attributes required to construct an HTTP request, and an 
equivalent response entity. Each request and response also includes optional matching rules and generators. 
Provider states define the state that a provider needs to be in for the interaction to be successfully verified.

Additional fields:

| Field          | Type                | Description                                        |
|----------------|---------------------|----------------------------------------------------|
| request        | Request             | The HTTP request part                              |
| response       | Response            | The HTTP response part                             |

Example:

```json
{
  "type": "Synchronous/HTTP",
  "description": "GET request to retrieve default values",
  "key": "163f8e0",
  "request": {
    "method": "GET",
    "path": "/api/test/8",
    "matchingRules": {
      "path": {
        "matchers": [
          {
            "match": "regex",
            "regex": "/api/test/\\d{1,8}"
          }
        ],
        "combine": "AND"
      }
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json"
    },
    "body": {
      "contentType": "application/json",
      "encoded": false,
      "content": [
        {
          "size": 1445211,
          "name": "testId254",
          "id": 32432
        }
      ]
    },
    "matchingRules": {
      "body": {
        "$": {
          "matchers": [
            {
              "match": "type",
              "min": 1
            }
          ],
          "combine": "AND"
        },
        "$[*].id": {
          "matchers": [
            {
              "match": "number"
            }
          ],
          "combine": "AND"
        },
        "$[*].name": {
          "matchers": [
            {
              "match": "type"
            }
          ],
          "combine": "AND"
        },
        "$[*].size": {
          "matchers": [
            {
              "match": "number"
            }
          ],
          "combine": "AND"
        }
      }
    }
  },
  "providerStates": [
    {
      "name": "a default value exists",
      "params": {
        "id": "32432"
      }
    }
  ]
}
```

Example of a multi-part POST request with an image, which returns a JSON response:

```json
{
  "type": "Synchronous/HTTP",
  "description": "a request with an image",
  "key": "a request with an image",
  "request": {
    "method": "POST",
    "path": "/images",
    "headers": {
      "Content-Type": "multipart/form-data; boundary=lk9eSoRxJdPHMNbDpbvOYepMB0gWDyQPWo"
    },
    "body": {
      "contentType": "multipart/form-data",
      "encoded": "base64",
      "content": "LS1sazllU29SeEpkUEhNTmJEcGJ2T1llcE1CMGdXRHlRUFdvDQpDb250ZW50LURpc3Bvc2l0aW9uOiBmb3JtLWRhdGE7IG5hbWU9InBob3RvIjsgZmlsZW5hbWU9InJvbi5qcGciDQpDb250ZW50LVR5cGU6IGltYWdlL2pwZWcNCg0K/9j/4AAQSkZJRgABAQEASABIAAD/4RguRXhpZgAASUkqAAgAAAAGABoBBQABAAAAVgAAABsBBQABAAAAXgAAACgBAwABAAAAAgAAADEBAgANAAAAZgAAADIBAgAUAAAAdAAAAGmHBAABAAAAiAAAAJoAAABIAAAAAQAAAEgAAAABAAAAR0lNUCAyLjEwLjE4AAAyMDIwOjA2OjEyIDE2OjM3OjI2AAEAAaADAAEAAAABAAAAAAAAAAgAAAEEAAEAAAAAAQAAAQEEAAEAAAAAAQAAAgEDAAMAAAAAAQAAAwEDAAEAAAAGAAAABgEDAAEAAAAGAAAAFQEDAAEAAAADAAAAAQIEAAEAAAAGAQAAAgIEAAEAAAAfFwAAAAAAAAgACAAIAP/Y/+AAEEpGSUYAAQEAAAEAAQAA/9sAQwAIBgYHBgUIBwcHCQkICgwUDQwLCwwZEhMPFB0aHx4dGhwcICQuJyAiLCMcHCg3KSwwMTQ0NB8nOT04MjwuMzQy/9sAQwEJCQkMCwwYDQ0YMiEcITIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIyMjIy/8AAEQgBAAEAAwEiAAIRAQMRAf/EAB8AAAEFAQEBAQEBAAAAAAAAAAABAgMEBQYHCAkKC//EALUQAAIBAwMCBAMFBQQEAAABfQECAwAEEQUSITFBBhNRYQcicRQygZGhCCNCscEVUtHwJDNicoIJChYXGBkaJSYnKCkqNDU2Nzg5OkNERUZHSElKU1RVVldYWVpjZGVmZ2hpanN0dXZ3eHl6g4SFhoeIiYqSk5SVlpeYmZqio6Slpqeoqaqys7S1tre4ubrCw8TFxsfIycrS09TV1tfY2drh4uPk5ebn6Onq8fLz9PX29/j5+v/EAB8BAAMBAQEBAQEBAQEAAAAAAAABAgMEBQYHCAkKC//EALURAAIBAgQEAwQHBQQEAAECdwABAgMRBAUhMQYSQVEHYXETIjKBCBRCkaGxwQkjM1LwFWJy0QoWJDThJfEXGBkaJicoKSo1Njc4OTpDREVGR0hJSlNUVVZXWFlaY2RlZmdoaWpzdHV2d3h5eoKDhIWGh4iJipKTlJWWl5iZmqKjpKWmp6ipqrKztLW2t7i5usLDxMXGx8jJytLT1NXW19jZ2uLj5OXm5+jp6vLz9PX29/j5+v/aAAwDAQACEQMRAD8A9VAp1JRmgBaM0maM0AOozSZozQAuaKSigBaKSigBaKSigBaKKKAFpRTcgVWnvIoFJeRF/wB44oAt0A1zkvi3TYphG1zGCenzZ/lWhaatBcxiRZAUzjOaANUUVEkyOMqwIp6sDQA+iiigBaWkooAWikooAdRSUUALSUUtADSKaRT6QigCtmjNJmjNAC0ZpKSgB1LTc0UAOzRmkooAWikpaAFopKWgApGYKCSQAOpNDHaMmvL/AIi+OzYRNp9hIolbhmzyB7UAa/iz4i6foe+CCVJ7oD7qnIU+5rx/WfGWp6zOXmu3C/wohwB+XWufnkaeQzXLlmJyRnJNaWm6fLeIRDZOc9DmgBLUh2XzH2nPXODXQT61eWliYreaRU4OQ/68U+z8FajOV81CF9Mc11UHw8nlsPLBdd3OSvSgDE0Lx7eWNvlrgzjOPLYkkfiea9P8N+MLPWYuG2Tj70TEAj8K8d1PwLrWlTs9oTIqnpt5rJt9fvNNvFSaNop4zw6nBoA+o0cOM9KfXDeDPF0etWyiTCSgfMo6Z9a7gHNAC0tJRQAtFJRQAtFFFABS0lFAC0UlLQBTzSZopM0ALmikzRQAuaXNNpaAHUZpKWgBaWm0UAOpaQUjNsQt6CgDnPGWvJoehT3TsOPlUf3m9K+abu+nv7yS5mbdJI5P5mvQfjJrhn1O30qOQ+XApZwD95z/APqP51x3h+wS4uUaZBwOBQBqeGtCN3Mkt0o25zivXtFsYII1VEAHpiuY0e2VQFUAYNdlpnyvg9ulAHRWkKjGFGK00jGKqWmMD3rQTGKAIntY5FwVGPpXHeKPAun6xA7+Sqy44YDmu74xUUq5+lAHzKv9o+C/ESR3GcKeHHR1/wA4r6A0PUk1TSoLpDw6A1x3xM8Opf6T9qiiBuLcllOOSMcj+VZ/we1aS402SykJIjJCZPv0oA9VopO1LQAUUUUAFFJS0AFLSUUALS0lFAFLNJSZpM0ALmgGm0tADs0uaaDS0AOzSg00UtADqKSloAUU2VgseT0pw61BfEi1bBwaAPl/x/OZ/FVxITnPTH1NJ4fu9txGnVgMH2p/xAtPsviaZVB2gfrk1D4XtTcXPmHgLyaAPUtEbdJk111kPnGwVw2nX0VjJ5kzBYx61uxeLtNgQSE/LnGQaAO/sn5Ga1R04ri9L8V6VfDbHOA3bd3rp7e+Rosgg0AXwTnFDZIqqL1N3JArOv8AxZpVg/lzXCh/7uaAH67CsmnShhn5TmvJPhJmPxPqEPPlKu9R6EmvSpfEdhqEUkCSfMynAzXM/DTSUgjv7r/lobqVPoFYrj9KAPR6KB0ooAKKKKACiiigBaKKKAClpKKAM8mkzQaTNAC5pc03NFAD80oNMBpwNADgacKaKWgBwpaSlFACio7gZTpmpRTJBlaAPn7x5BcX2oanEiDEM428c45zWf4Tg8m0ZWGGBwa6jx4sllq0vkgr5xDOR15rB0hlDTqDkhzn86ANZ7USgO33F5xisi6tNX1QMLZFjRThUPBNd34fkg2ASIp7ciurj0zT5WEgiQN6haAPG7rw5rOjCKe3YynGWB7V2vgbXby6uVtLzKNjpmu6m0OCaEtJHkAd+a5CG3gj1xXtgqsr4wvGBQB0vi6G9TTw2n7fOK9WPArzKy8LajqmqtLdTrIm5dxDEMo78dD+de1SxLcWsfmAMpXBBqtFodrDJ5kMIRj1K8ZoA5Tw9o95ZyPDfICqEGNxzn61seD9Nm09tUEjKYpbySSJR/CCxPP410rRqkG0LiqOkg/6QxxzIQKANKiiigAopaKAEooooAKKKSgBaKSigDOJpM0hNJmgB2aXNMzSg0APBpwNMBpQaAJAaXNNFOFADgaUGkApQKAHA0pwRSAUuKAPPfHullpI7zGYyDG4x19P615fZsttq0sSfcPQV9B6vaw3lhLDMuVYV4FdW0Vt4pmhidmVSeTQB0lhcmJ1IOOa7nR79XUbuTXncB+YYrq9LLJGDg8CgDrNd15NN0xnXBcjgV5xYa5p6X8M8kwWZvmkGe55PFbOp3H20+VkH8awV8FvfXGBII8nOe9AHqllrWn3enptnUs3AAPJqtZa3Nb6lJZXIII5QkY3Csvw94cGjqFllDEHIZjWlq9tHchZ0fbPFypHcelAG3Nc7kJ6cVX0Z1e1baR/rGJx/vGqYkP2Us3Hy0vhsYsVbn5yW+uTQBu0UUUAFFFJQAtJRSUALSUZpKADNLmm5ooAziabmgmm5oAdmnA0zNKKAHg04GmCnCgB4NPBqMU4UASg0oNMBpwoAeDS5wKaKjuJhDFkjcT0Ud6AMPxXqEVrpr+YxUMOMdTXhmqGO01uGdTzJktznPqa9P8AGKSyW5aR2eUttUBuB9K838T6Y1qbGZQWzII3PXG7v+eKANqA71DDkYzXdeHxHcWRD+led6eXiiVJDkr0962bTXm0xCpbCk0AXdR8NQ/aWmE8uGPOHNLZ6do9m+6a5m45OWrPh8QveoQp69D6UwWHnygM5ldjyeoAoA7OG30O7CKjyHPXJrWTRrKG1BhQDHINYWiaVDa4EjDjk+3tUmua2LV0t4mHPKjPFAG5fNsgCpjc2FUeprXtIPs0MSYHyoAcVm6TC95Il5KuEUfKp9fX/PrW7QAuRRmkpKAHZozTM0ZoAdmkzSUUALSZopKAFzRmkooAyt1Jmm7qM0APzTgajzTgaAJAacDTBTxQA4GnA0wU4UAPBp4NRinZwM0APLgCqN1Ltb5mAOMjPapJpWAyi596xNRhmuAS7HkdulAGNqc0VxKFRhIQ25n68+1Y2s6WL/TpI9vOAyn0I5Fa1rb7pHTGNp5FaBtwFxigDyyK4IXY/wAsiHaw9CKrXszzRNGCN2OOa3fGOlnT7n+0oVPltjzV/rXOlklxIMEGgCzaS/Z7FVQrliMk+ntVm11iWwuvKdsjHU/TrWRsl8lFyF9F9KjmixK2ZN28bunQkZxQB1kXiqXLojcHufypmZNQvIG8wlweh5ArnNHhNzPtxznnJ/L9a73RtPWOSMCPcyMMe5oA9V06EQWMMfI2qBzVrNJADLbJIvPGDR3oAM0ZopKADNFFJQAuaM0lJmgB2aTNJmkzQA7NFNpc0AYgNOBqIGnA0ASg04HFRA08GgCYGnA1GDTwaAJAacDTBTgUDBWPJ7CgB6gscCpHgIjDfeJIAB6U+xic3jM4+QD5R2FXCm+4APRTxQBXmti0OzuRyfSqdxbL5YCjgcVtOuRiqRTO5T1oA4u1iC6zeR9yFYfrVuaPjNOuofs/iOF8cTRsh/Q0ancQ2No80xwFHQdT7CgDI1mO0fSpheOiRFSCWryCe2On3GYm32kpOxh29q1PEGo3uvX5L744FOIo88D6+9W9MjS806TS7qMLJklJMdDQBivEzIHjztUDOPX1qjMjrGwznLZYdM811+k6VO0bRtERIODkVqQ+DFkmDOpbJycigDA8OaY07ZUbUGCxxXeaBsukkdV2hX2Bsd/Wrf8AZVtpuliO3jxKwKouOWY9BUmkG38O6I5ngmun35fykGc/jigDr7K+toFjtpZVjkk5jRjywHX+YrSeMMueorzW7u4vEumW19BA0U9nchApPzKrd/8Ax0V6XCG8hAxy20ZPvigCs8e3p0qOrpXPGOKjaAMMr1oAq0mafJGU57VHQAZopM0lAC0UmaM0ALRmm5ozQBgg1IDUAapFNAEoNPBqMGnigCQGlaQIOozUM0ywQPK3RRXMLO11cvKScZ4570AdbG7yZx6447Vo2lsPNDOMj3qnpkDJbxq3PGQfWteNdtAC2Dfv7mI8+Wwx9Dn/AAq0i/vGaqVgwOoXv/AP61oKMUAB61VmXbOp7GrR5NQXI+daAMTWbCWWa3niA3RNnn6Vk6nor3m2WViwByF7CuymQMnSq6wh4ipHSgDy+/8ACqbzJGnB5x71mS6LN5wMaETKOmPvCvV3sFLEEcGqtzpHmR5jAEycofegDh9PuJrKYG4tNynqcEEV21k8M8CyooAYcVajsbbUbEMY8Bx+INZtvbyaVctbSH902SjUAVbGOTVPF7Kw/wBHsVDYxwWPT8q6b+zoArJ5YIbkg1meDoAYL67JyZ7piD7DC4/8dro9uWNAHMr4cS2vHe2G2Gbh07Ag5B/nXTYwKNvWloAQ9QaMYNHVfpSjkUAMeMOhGOtZjAqxB7Vq1Uuo+rCgCpmkzRTSaAA0maQmkzQA7NLmmZoBoA59WqRTVcGpFagCwDUitUA55qQGgDN8Q3BSzSIdXaqOnwfdXOMDJNSa2TJfQKeiKT+eK0NOtQdPuJmHVcCgDqbQL9ghPY4GavAHHPUdaxNKmMmgxHOSp/ka3kYNEsg9OaAM/SW3X1+f9sD+da4PNYuh8y3zdjMRWupy5oAcTiSopjmVRT34cGo/vSk0ASjlKijGJGHrUqHnFMIxLmgAZM0gAIzUjdab0P1oApRr9lvWX/lnNyPZqmu7dLu3eNh8y8rmluYvNhIH3gcqfQ1JA4liVyBkjn2PegCl4ftRZ6VFCOu+Rifcux/rWn/GagsV22wH+03/AKEamb7/AOFAC0lIDxSmgAHWjoaO9B7UAJ3qGYZQ1L61DKfloAzScEim5p042yH3qLNADs0maTNJmgBc0ZpuaWgDnAaeDUINODUAWVbipAaqq1ShsDJoAzL/ABNqIVeSABXRGMW2juvT5KwdGiN7qLSt0+9W5rUnlWTDsRigBPD03/EtEZ5G410NtPHFiJ2A3dAa5Tw+T5CD/brc1UD7ITnayjKkUASaAjxxXQcEMJ2Bz9a0opMTsp71keH7trmwEz/ekJY1eL4n3e9AF6bpkVHGck08tui/Coo6AJCdpBp5IbBpjcimqcGgCc9KYeRTgeKSgBOtQQ/u5pI+x+Yf1qY1DL8siSDscH6UASWTbrYH/aYf+PGpZDgimQIscIUdMk/mc0M2W+lADgeaXPzUzOKAeaAJM80UzNBcY60AJnrUEp4qUtxVaVugoAr3I4Vqq5q5MN0J9uapUAOpKTNFAAaMmkzRQBzIalBFQhqdmgCcGmXUvl2shz24poaoL4+YkcI6u1AG54Wtgtuzt/FSeIjstmTPfitPSYvIg8vHTpWX4p4hB7E0AJ4dXdbjHUNWzrrf8SaVvRTWR4aIFqzHsa2LvbfabPCv3mUjFAGV4fu1jsIgTwa2DOhbOa4XT5pE0VOoZMjFVm1+6jBCk5HrQB6lDMHi4NPU4rzjTPE18s6FkLRk4YV2aampQNQBsDkU0is+LVYywBzj1rQWRXXcDwaAHKadmowRTwaAA0xuVINPNMNACW7FrYfUj9TTiVU4ByaihBCbFONrc/nn+tTOiZBFADAc04H5hSEYprEjBHUGgCbPNNJppYnsBTWL54C4oAHIxyKpu2JMegqZ5CucowA78VSaQF5HB4oAnYjy/wAKpHg1NK5FucdSKgZdgAJycdaACjNNzRmgBc0ZptLmgDkt1ODVBu96UNQBYDUtrGbjU0XqEGfxqEPWn4fh3ySzH+I8UAdHCNqiud8SyHy9h7GuiB4IrmfE2flb3oAveFQHsZc/3v6VcjdobtkzwelUfCL/AOjzL3zVi6fZdBsjg8igDmw3lm+iIHyTEAVhTsjOT9a2tSYQX9+M/eO/881zAkZ+nQmgDoNMIYbQBXR226QD0xg1yWjXMVveIJz8jcZ9K7aGa3x+7YGgC1FGgHIrUt2xEBWWjgnir0TfLQBeVqeGqsj1JvxQBNmkJqLzAwyDRv5oAkiwC+O55/KmysVwc0sbfe470SEFaAGJMGyM0pPFVjhZcipd3FAEwbgc00tUbSFUz7VQ+0sJPm6GgC5cS+XETWdK+2IL3Y0txc+ZN5ZYADB+tV3bfOq/3eaALTnKIPXFJLjapFNkb5kHoM00Nuhz70AJmjNNzRmgB2aM03NLmgDjM4o3VFml3UASljt469q63RoPLsVXo1cpaxme5jQeuTXaWpCxgZ5FAE+DXPeJIibbd6Guj3DNZusQCazkGOooAyvCMuLh0P8AEK1dQgSUsc7WHRq5zw3L5OpiMnBziulvwPMbIyCKAPPfE955OoSLnlo0/rWVbyfuFrP1rUF1DW5DE26NPlBHTircBJRVA7UAWC5J61oWeoyQFSXOKz0t5H6CrtvpMsrZYELQB1elass7+XzkDrWpJq8cJxu6VyQaPT4isfXHWsqS4uLiXCljk0Ad6/iAou6IBiCMitiW8DKuDjIya4vSrFwqvN0zkiukg/ecnpQBq2z7mb0qwDzVS3+XJqxnmgCeNvmI9s05zxUKSBSc9AM05nBUMpDKfSgCtM+xgakjkDCo7hdy5FUUuPLfac9aAL8smY++az5WB5FWJJcEZztIyDVKSaJARuXn3FAHNaynm69ASTygxg9MV0FkSxyTnt+VY17GH1GCZMHCMOPwrZgP2a1Bb7+KALDSbpn/ANlcUQNuts+9VY9ywSO/Vuant+LQZ7mgCTNJmm5ozQA6lpuaKAOIDUbsUzNGaAN7RbSV4pLlE3YO2tGOeVHyfxFP8K86a/u5qa6jCTHjg0AWY5A4BFPkAdCp6YqghMfSrkMwbgjrQByU8LafraSKCFLjmt/V3dtOaSIZcoQMetR6zZGZN6DJHNTRTAW4V1yuOaAPJI9FMN9LGFxhsH2rftdPhiUGVwPxrjtR8WPLrd3JbKDA8pKfSpodVuLwquTzQB26X+nWrhQnmNntU9/qkUaAQgDcOlc5BGtrCJZeZD90GozI88nJ5oAtNI9w23J61uaXp6phmGWqlYWgXDNya34VIQBepoAsIS8giQfKOta8SbEAFVbS3ESZPWrRfmgC3EeKnzVWI8VNuoAmQgyEEdqcUCphcKB2FRK2GUetSE8UAITlazry3zHvUcg1eB4qKU5QigCOxZZodkmPxrG1nTvLJeMcH0q2shjWQKecGq0erRzQtHMRkcc0AYCTyxojJjcpI5q9by3M5w0gZj29Kzr2fLEW6g5brRZWl5LIJPOKfQcUAdFLvisyrnLHircfy2yis1t6COJ33t3NaQGVRaAEzRmk6GigB2aAabmjNAH/2QD/4gKwSUNDX1BST0ZJTEUAAQEAAAKgbGNtcwQwAABtbnRyUkdCIFhZWiAH5AAGAAwABgAkADFhY3NwQVBQTAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA9tYAAQAAAADTLWxjbXMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA1kZXNjAAABIAAAAEBjcHJ0AAABYAAAADZ3dHB0AAABmAAAABRjaGFkAAABrAAAACxyWFlaAAAB2AAAABRiWFlaAAAB7AAAABRnWFlaAAACAAAAABRyVFJDAAACFAAAACBnVFJDAAACFAAAACBiVFJDAAACFAAAACBjaHJtAAACNAAAACRkbW5kAAACWAAAACRkbWRkAAACfAAAACRtbHVjAAAAAAAAAAEAAAAMZW5VUwAAACQAAAAcAEcASQBNAFAAIABiAHUAaQBsAHQALQBpAG4AIABzAFIARwBCbWx1YwAAAAAAAAABAAAADGVuVVMAAAAaAAAAHABQAHUAYgBsAGkAYwAgAEQAbwBtAGEAaQBuAABYWVogAAAAAAAA9tYAAQAAAADTLXNmMzIAAAAAAAEMQgAABd7///MlAAAHkwAA/ZD///uh///9ogAAA9wAAMBuWFlaIAAAAAAAAG+gAAA49QAAA5BYWVogAAAAAAAAJJ8AAA+EAAC2xFhZWiAAAAAAAABilwAAt4cAABjZcGFyYQAAAAAAAwAAAAJmZgAA8qcAAA1ZAAAT0AAACltjaHJtAAAAAAADAAAAAKPXAABUfAAATM0AAJmaAAAmZwAAD1xtbHVjAAAAAAAAAAEAAAAMZW5VUwAAAAgAAAAcAEcASQBNAFBtbHVjAAAAAAAAAAEAAAAMZW5VUwAAAAgAAAAcAHMAUgBHAEL/2wBDAAMCAgMCAgMDAwMEAwMEBQgFBQQEBQoHBwYIDAoMDAsKCwsNDhIQDQ4RDgsLEBYQERMUFRUVDA8XGBYUGBIUFRT/2wBDAQMEBAUEBQkFBQkUDQsNFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBT/wgARCADIAMgDAREAAhEBAxEB/8QAHAAAAQUBAQEAAAAAAAAAAAAAAQACAwUGBAcI/8QAFAEBAAAAAAAAAAAAAAAAAAAAAP/aAAwDAQACEAMQAAAB9xHiCIQhACIIRDwhCEQRCAMOYQhBEIQRBKc8gLA2psyYQRBEIIDhEIQRBEEJhD5VHm1MwepH0GTiCIQgiK8AghCEISvPik2p9AmzIzwcvz2IIhCCIJWgEEIRwh54seDHux6Ya46TiPmc+lQhAIIglYNEEcOHBHHzgeQn0GeiGtOojPCj28IRCCIRVAEEeOHhHHh546bAqz0E9DPKz0M9LCIQhCEVABDh48cPHnl54mevm4MWelkhGWoRCAIAinAEcPJBw8cUx8xGxPUzx89YLE7zRiEIaIAimAEePJCQePOc+Wi+NqU5vjvJDUiAAAACKUA4ePHkpIOMefOZoDQHKbY7jfHWAQBCAAowDyQePHkpEYEzR5scJMWZ6Qe0gEAAhDQmdHEhKSEg8ccxjjqPLDAHOXp7yelnMNAAaIATMkhITEpIdJagMccx4MRHebM1xpjTnMcw0AgCMuSkpKQFmXZ3iKIzRhzmPQiQuyxLERxFeNGgEZYnJjjKg9NEXA0gITnOUiAaQIgDClIRgBGTJycoCyPQzjLwiEIJGMCdIRAGFMcg0AjIkx0FKXhrAFuNHjh4w5yYeEIiEqyvAABjyQnICc1xWmhO0cMJwEBKAQ8BzHGVghojFDhxflKbwwZamkLUlCOGEQ8BIRnIcxXgAIwo4YegGINoY8w5rjWmjO0kGjRoRDSrOUBEAR5+PO43h52bow5gzam1Lc7h4SQhIhxXlUdIiMQTzkcao1ZgDWHlhmiU1BojtNEW5OMK4lKgxxsADhBPNhxty9KIjPHDvNCQHKbs0pYjx5VDDNnEWx2DwhPMzvPRDkO8oDycri1JzbmjO87iYcRFOVBkjTFyIQj/xAAuEAABBAECBAUDBQEBAAAAAAACAAEDBAUREgYQEyEUICIwMQcjQBUlMjRBMzX/2gAIAQEAAQUC93X29PcyGVr4yG99Rh30OON0WN4qp5IQPc34fE/E8eBrZHKWspZr0zN62Bt2YJ69zFlwvxkAnFI0ofg37Y0quXy0uZyHD2CCUsfRhAIYRFp8bBYHijgZqrcA5579f8H6m5F62JwMQnNiwbbQf0h8aqcN4vC+B46/B+qe4lg9xWcQTbKRA6hkZ2Z2UhiK4ioFc4yb4/A+oUTTR8NQbFPrGIw5WWXhviW1JPxBat0IKUGXmUNKQMz+DxtCXhMcX3cOURx1sbCTZOvBXyJxBPBWoR11J3v+w/s5ai2QpTCcGXozOB18i8Ve7lLUVvC5eXK1cfYmryDP+5e4ybyWJGjizFgjzlTtLkq9gK1MbhHG9tk0cvVqh1LnvNzzkpHVzFMwjrSOwHxKpN9ssTTGs0OWK1ka1dq8Psa8m8uqlk0HIXBme7jvG1OqbKM9LTXygOLNTSLhCruyPs6rVMmTc+6mq7juVenaKLa3FWLONxcZGMmFsHX1k4ei6MhRJ209jVapk3OId0zRs8ko6LMxbFdIK0OUzlnIWDoCNl+HpJ3HCfp2Mr3KWJj4cOz0CFFE7J/MzpnTJl4oHeOJ5FFoFoW7k2rXavi6d3GPbCfhzYdbF2IVjJRnGhC97L28PXuQ0Kz14lorAeTVapnTOmdX7Hhq2KDqTxhtYS/cGX+Rt6GBeGZdBq89qhsPh6PSrp5C7sfYvIzpnTOssW86LeHUfc4//Vb5d9Gj7L4N2Rg0gQE5DUHZF5HU/wDLXyM6F0zq23VuX/swxSsxQd8pGXrP5/z5T8v4zV31DXv/AIn+CU3cfIzpiQksZH4i7ly+9UbWDEzffE9JZO6/z45yNqIOzBqteTujJfI+TVMSI9seAi6dfL/3KvYKtnw9n9chZVclDcAZG5Nzif099VuZbmUxenctd3k1Wql9bVR6ceXPdcld4WvbWyEgtvxrNEo9xqub7GdarVAnk0fVM/Z31Uu3ex6xn2bnuTEqQda38Nle12wWtXJyMMzE5rEW4WCOUSURdhJdRtdyF1Lo6AuzntEbD7muBMI9ot2orVarVarCxras4G2aA9+P4nsNAURfbE9Hx12XfPnBiejkpJ5wkci1QOjQSbXMuxksTHtTn64n1i56pu71pBriMm5Ziv1q+Jl6uPzvUt5mrQM2hxsSsWQiVWm9o68bAo20TOu6f5sRuofvhdsywHWsxwnFJuUP/DnqqH9y5E2sZbF2lCnG9WbibMRUszHlpra3+HatC8pVw2DXiaIQLV2dC/cnROt/SK3aht13OR5AKVoBb0c//8QAFBEBAAAAAAAAAAAAAAAAAAAAkP/aAAgBAwEBPwEcf//EABQRAQAAAAAAAAAAAAAAAAAAAJD/2gAIAQIBAT8BHH//xAA6EAACAQIDBQUFBgUFAAAAAAABAgADERIhMQQgIkFREBNAYXEwMlKBkQUjQnKxwRQzQ2LwUGOCkuH/2gAIAQEABj8C/wBENTaKq0184V2agx/udrRqm0UymXKZN3bdH8Kbce0H3VhrbTWNzpefdhqvoIQyXyymDaKNTAPOJs9apiTRWfUQMND4KpWc4VRSxPlHrtnc8C9IKu0LiPSDDTX6TIWhD0w1+ojbX9nJZhmafWNs1Q/e0+vMeCWgv9d7H0EUtnPSDtIlLBw0qxuB68vBbP8ACDa8ULpDnaZMPrLGazMgT7Mw/htV/wCpv4KhT7vGz4yM9CBKpbJpjJYqNEU6ypVp0ymEYtdYuzbSjIxyu0U7MneM30Ebaawepn7gOvpPsyqnGoSorljoDa3gqdVBxqbD5yoDm2V4FdAZZEssSmgCuMxKfeWItzn3agLKVuhPgqlPny9YUZSjWzBghPO0dyoq3bFeBHo4F0JMfZqrFivut1EQf2HwRMWs2HC/CLRfWIdnwt+YxS1LZR5sZ91/CC/wxTVKmwywiVKvwgL4KoAcFO31myVTzbCR8N4L+8IlLnL49dFE+8qaecagnEQLa84F56k9T4E52EwKCTbU6CNT66HzjI3DUXJhO8PSKytnzjLeyypVItYXz8COV5bWUR8ZI7BttAX5OOomPUAT3bYc7zMX/aK2omXt1HLrB0Xso1fgqAx6lQ4UXUzgxU9lU5Uxz8zFqUP5FXO3Q9JwjCpjimcdZxYTZaNWqe8IVbKpt9ZXpbTUNVqNUoHbUj2uFXUn1mcCDQpeEwiMnMiKahvblMVMXhNNMaX908pY08DrqJtLN/J2de7X1Ov+ec7t09D0hDG7E3J+Vu249izczkImLU5y0Uf7Z/UdhhjDsD/hfhb15GfxFIcQ95RzErVLWNSu7H62/beI9hRTlmTKFTk72jjmDG8qQ/XsaDtKnQzP3l4TLDQM3671/YIo1tabMvRxO806yu34TTSx+sYb35p/yP6+2q1eSykvK9402gX4VOFewbvmM5lpvHfY9BL8zEE9Ztl9VqTNrS6ODbWa7tvMzPs1mvYZfeCfEbQekUymwm08saK/7Tyl055S5MsTuH1me4AMoxg3k6Ln2ofKJU+Klb6H/wBnlClTJxMt07mcZlcM3O3KAec+e8znn2I0pelpSVcvey+kEvAAOHqZYRWvwZ3G7Y9l4wtbjMVemc+e7YaxVHLsJGozhXmsc/01yXs4nW45TBT0l20gRPn2nCbHsuJrZhLMBGyJe8drZ9YN2j+YTFaeUIlVbcJndU0xYUGLyP8AloFTKWveodTLnSZTz3ntLtkwlqa/OWq4emUUbn//xAAnEAADAAECBQQDAQEAAAAAAAAAAREhMUEQUWFxgSCRobHB0fAw4f/aAAgBAQABPyFLjSlL6qJO4n67waHwUpSl9SpA5svstxlUNsh4gqb1JNa6KGccsl+YKsv86UvrxcR1dOrGrzMNvRLZCI3zWGxQx6MbH2ZQ46prBeUOIviqP5Pmuv8AJ5l3P8aUpSl9CPLYtkWTUla1Y2FiNhtBQoLQRCexCCNq3DtED03nUqHRVuBf4UpeK4ofc1Bq3zT3+iMbxzoMYrwJ2HwYlEzrRKPeR6nfdfcWn+DY2UT4rgh6Jkr1ZpqMQk5h5okSdfQabIPvcUqnVia05HRn5I0vXRspSlELgh922EZBPqMdB4FHsPHh0huLK8ld+HXhvCLVyOXZ1WexwObdpPbHLXQWE8vBY7nUl7i/wbKUTEIQhGVJ9/KP+GfHBmfOZE9t61DKs7VbD7jJiWrEMrcPYZvVETwJDoTS8T8r1UpRilExCYhMTHBlK3yZXyY7coQj7HN0YU194mq2iEXGXDeyMjoGLtsvC2rXyvRS+hsomJjDCEIovRDW8rYodJvRhbLmsqAkrLct/QmVt1ZMeZ1rZiNauU45/kWnG+lsohCEMLhmYivf9UVVwseD5+zkZqZIYqRGARo/LLFuG+hcvgSJoHYMT35o1ZoUpfUowmJiYmIIfcCpQ5Q9A3SL9iOxsfuWo09AxCHtPdML+ow+3yrW1mnggzt1S/unyPa7YXVwpeNKUoig3AwhMaeEsmGxtlxruwnZPgKKQoWeko/cENE6EE7wDXp/czKOXvo1v0Tt0Uq0m5K6IN1LjeFKUQQYYQubcXNj49dNv9RBJanlmtWlrJEb120f2JCoVnIp9QhGgKsbfEd4VPKm7Msn7WdF46maSoXJcINaK4oc+ZWoyiyik48MbKUpeIMMPbWYqZ0JyJA0Ncmml+Tvp8PI/FBWNq8vpRDka0T+hTBNDaWzH1PF1RJdhlo9d4UQgREonUIcTlu4r4Hsz4E52mUbG/UWRRwjmsV0ZmGqO1v7PHBawsgW23uO7U10Gj1PE/wXsKwr7JFE7+kn8KLAhLULQSyYhDY2UvoK/oykHVJQ+I/XgvYarWnym/Q9bhbChOCaULGICf5H+wLgRAl5icT78XuNgx6nBSl9AOraHyFSjBKk4gwbLLaapvuB46HuLi+DTBpPZY+6K76PzHlLwMGwJXoy8aQ4haMsx+B8mYoMwL06D3ehQ9ln9mUGphBOy1DGUWpBb6jb+TLPDWPexI0HNDKXguE6+jKrLmjNcq8CR6B6mGWJ5NXeWCdC6Dewao04Mgu9zXljyROw9GUScZ6jc4kvKkn9hYdJyjVzkpS8KNJtViFSmg5rbjSsRXoYMoeddfwHYurQl4mgXIV0NRE3MfAxhd/TBdDskgpI8rqOiLDLS0OtTZ9CUpRdXA5mTQng5H5oq/E1yUNlrgmPuHkUVXRo1OhC1l0SRN5fD8wzO3A0XoIOjtKMe9h4URq+ovoKPe2cHLl4EtvMoIGbtLW1LF2yHkONka8gofTAizJVyTULZ7tGoZnkZdeCl2VHYmCHRRo8JkuTUVH1jYi8FqyM4kZWojZEmmoOOK6vPC4ZRNL3+aI9URlVIrV5EDTh8k1vMfpSBTJcjAJvKYqUfllHK/KLXNDG6XeT/ZF3DRJYVdF5rDUNOvkomXgY5VEjnBmOR4oNVkVdT5wmLb+mHjDrkjPSR2ugtAkpcvCLB6teAYCfetf7uaJD2E7+vLXsND0hkbpsyi/csDyvjhSn/9oADAMBAAIAAwAAABCCAACCCASQQSQSSCSCCCCCSAQSAQASAQAQCASCCQAASCQSCSCCCQAQQCSACCCQSSCQASSCCAQAACCSQSSAQCQAAASQCSAQASCCCACAQACSASSSQQASQCSSSQCASQSCAAACSCASSCACSASSCSACSAQCSQSQQACQQQAAQQQSASSSSSSQAAASQQSAAQSAQSAQASQQASASCSSQACASQSSSCSCSSAAQQCQCAQQSAASQSSSCQAQQCQACQQCQQQQSAQSQQQASQAQQCACASSSQQSACQCCCSQASASASSAAQCASSAASASSCQQSACAQSAQACf/8QAFBEBAAAAAAAAAAAAAAAAAAAAkP/aAAgBAwEBPxAcf//EABQRAQAAAAAAAAAAAAAAAAAAAJD/2gAIAQIBAT8QHH//xAAnEAEAAgIBAwQDAQEBAQAAAAABABEhMUFRYXEQgZGhscHR8OEg8f/aAAgBAQABPxAyGJcPQPQu5YwbgwalAtYJ3GCEIMuXTLhDLfSuoYeoehcG/Q3NQsK7sTpsXYzDBpQE8BEt52nEIkVaAArNbG+fuo7VIGKugRLwKnNQnmIQh6XiXBi3EuLfMIYIG4MuEGcRovMbB/SH3xM6EwaLoXYwHmWmjWA6ar26QPToALFK7HNuL97uFamsOw/FnkjkVScID8mjcxY2sH2JYEfx6DLly5dQYehpMoVYQZS8wYQ96VXmlXLwHVlrauCluH9etssBkTYOP97wYoNCEP8AD0MuAvpKjR4YRi+MvhOmlp7j2fdcLOKPdKp9rzFYempdwZfoRhkgYMXoei7IBK2vvrfbulwUIJzaf95lsa8CkIvA13mSGyIKfiLAA3SXKh4Kw3hDlNP4jx8S9S4sH0GX6gZSyXmFkUJmR/PcW8UO+K+GWyctC8Jtfp953BM0JqWiXmpF8dYgQQ28xGAGsQdAC8ZAnUkOQZX5RvudY1uKwb9D0GXiX/4IggwisiuBMSGIBKqCChyceF6wRrwiUtYzzgGy0GO9e1ss9aNvqkXp+u9Kzittrrbl95TWKhDgWy12DNSuiQqHKzNUMkWgqtww2cI7OQpOioKD116LLlzJN/VO/SvUXyvJUih5qfEt2VorSB7jZ5h1QqiuqzjMDju1H7G5nAAER2tdCvB2gI2GNuvaHlh0dVxmrluAChmgfC5V6u5cfQPQ7Qlo+8oqNiRPMHPI2AJa9gEDGrSVi/OeTEQq8mpSY4nAuo5EzxQAGDgMFm5U2YKkTyfKVGHngF3I4vh7j2i3MJGasvuvr1uo+gtRZczQi2V+gLuMlZSKIC1X1qV5fYqA4p5FbxincqFqDbBfmX8PIOcTVRwGbOov4lBIdDorrnWoNCQgXVmVs0k1gob2LevYfUdhxFCLGFlxYvpkvHqv1g0WtB1jhRpqKjRfIhq33ya7SYFCGJQUAf4IrVAVHFnPvLXC42dYiDSvJiZLWZrUI1pvmESuyi5xjVzQTAGJ3XTv0IQcpMbF/JroUcQNDUVLxtLiy5cb/wDiycRmNKodZtK1caNIZkVR5Uov+1CKrvotJs+4B9pY2uF17x0cI8iQJBSKijW+uwdTvHY+5Yix4ZwovI6EmxICNHW/hroMTGxz9o2d6toUBezHBiNJlFZfpcJm0zyissmXcR1gMUALcHWKVCgbl7G2dcdnd9Ha39QFFob1Yx+IKMArUUgIP4qdT8RWSKraWi65Rr2vtReEcCqXNnjF4eTVhbGsZCzaWwxWM3p5lGgabO+WQtVad0fJpJCyOSN1V34jHEaxc+jaeUz+lb6buYCANpQQC4cXTj+n6lZEVnKmLXxfzBAGHuJq2Ku9/FC82FaHba8AZWiU0BYDHCG16aMb2rYI5FLmKxsTGnWI8DfG9ovcrl670QCIBrk9C6BV2ChXqA03jC71hBVu720rBK7zjCleVj0TpDdFGrOP+Qi0SBQIbGUTaMKJky+WTPNEVE6I1l1QHMIBphVsVe/MFDHVMIleawCtwENosSIooTQJp+iW+TbiqKWuVty9YOItTxTy7Qkd3cTsmU2XvkR5lDeLGHF5OuPjMLEAMVS+YfBN46pgPwj2gY2hVV8pn7weQwwBYmXEoGrgPqMUYuKMxlGsy03MB+4TEvIDci3b8DFLYSg8DVePzfWXlGO057/2Vp2596owKNSnZGcsbSGoI4RtTWWGmGfplWP/AKDe/RDOj8Abo61LKradU3jA/wCwl4/1zBUnuGYrz2QRF3B4wceJY+gwUZnl5jcTmMHyF2KfkxcIkEyaCe81+g74jH1AbWodv8ntKx0xNKzkCUQ5Jv8ACVMyGzJAUtlGzx3NzNXkasYma7gjzBRSI0E6B3H3Fws0niYvQkopBSNWI25j3x9kud4UpSXcwthcHFp/koNSTqVMCqBcVHp6NlF1PkfqAi4Nkdwscx0DwXKk9mGLT7QyeDmVUtdrbj5L+CCJj7QL+JUHA3PmWKYmYyk5lnM5pe4suzvBmK9F8ksq2MBXFGdY+hGKf/AIgGRqxd01EWiwabJ8qdB20ypo7oxuzcDJ2XzOAC8h/wAsmoHuda/awSUVbNi9DEKavwwChEEl2Ackcx0H4lhp4mWPRcw4mTM68geax9wg4W7dEshK1dbzBO4cp1xLNEAPJl/cdoLqGVCgcl4jYZPMIObGNhM1MWyXELdRY5jxiX0CvWkvUbqsyo6AJewxiic0Cx25zX2hFyV/EaqAxUr39L05hhAmb9oF7js7A+12/iGLKnwMQRsAod7ibcTlp+Jo6bHCzQsAfgN0/iUKb83tMkRAaXlqECHgKWC2G8JhWVrsexj9+8oVQYAs1uCIArGKzGAjbBcqJDe12Y11z2ilrVT8EKm7owDVV/Zn6S0yXl2lDmKKlXBy4Pwy6HcS9zpARCB4Ms0VcShwBrIynoShiRW1/wACNwAvDSU8QwUIGJusyuUSiOYLDp1X5YIBSZhEl3RUQAKLj3cx4rvJ/IH/AAAdmhU4zcoBi/IyX+I7a6KMlNwFQo413jXuCBMUR2Br9zou+/aLXuRv4hvBoEsSq/UHh4YUJY7YZbLr9IpCE5JdKFXKIBJbadZYak+we91NOhW+8yDeGXs9Cj5tjzpMHTFVOYx4VbCs1Rz/ALc4SXSdIDERAptFPuSpm16+GsfliqN6mWG4l3A6hcwxGyV1C6Je0AEbeWWJEWUwq7cFbxuDnArOv9cvRSpDWE9VsvpXSMuJq1ohx0ms0h0jDZyh0jJ3YV0AsfiUKiwZlUasGwbUs5Gou+xdaYokFadYLzBhwnR8fth65peCoENX4KWvvcKfIqezoRE6VaF+Zkh8pwrUvPWb4b/UEinFG4KK1rDoTCC1QjzcRlSFMDInxMX4KotrPke8Ip40xUu6V1l1/aNi5Xmae6kPub15k9ZkxlZmNDObwcH2mbK2/iCHTaMx+QoDowuuBwbOkyrQDbQ6cx5cVCx5uVC0bXYIPSYcwrP/2Q0KLS1sazllU29SeEpkUEhNTmJEcGJ2T1llcE1CMGdXRHlRUFdvLS0NCg=="
    },
    "matchingRules": {
      "header": {
        "Content-Type": {
          "matchers": [
            {
              "match": "regex",
              "regex": "multipart/form-data;(\\s*charset=[^;]*;)?\\s*boundary=.*"
            }
          ],
          "combine": "AND"
        }
      }
    }
  },
  "response": {
    "status": 200,
    "headers": {
      "Content-Type": "application/json; charset=UTF-8"
    },
    "body": {
      "contentType": "application/json",
      "encoded": false,
      "content": {
          "errorMessage": "",
          "version": 1,
          "issues": [],
          "status": 0
      }
    },
    "matchingRules": {
      "body": {
        "$.version": {
          "matchers": [
            {
              "match": "integer"
            }
          ],
          "combine": "AND"
        },
        "$.status": {
          "matchers": [
            {
              "match": "integer"
            }
          ],
          "combine": "AND"
        }
      },
      "header": {
        "Content-Type": {
          "matchers": [
            {
              "match": "regex",
              "regex": "application/json(;\\s?charset=[\\w\\-]+)?"
            }
          ],
          "combine": "AND"
        }
      }
    }
  }
}
```

##### Request

Represents the request part of an HTTP request.

| Field         | Type                      | Description                                     |
|---------------|---------------------------|-------------------------------------------------|
| method        | string                    | The HTTP method                                 |
| path          | string                    | Request path                                    |
| query         | Map[string, List[string]] | Request query string                            |
| headers       | Map[string, List[string]] | Request headers                                 |
| body          | Body                      | Request body                                    |
| matchingRules | MatchingRules             | Optional Matching rules to apply to the request |
| generators    | Generators                | Optional generators to apply to the request     |

##### Response

Represents the response part of an HTTP request.

| Field         | Type                      | Description                                      |
|---------------|---------------------------|--------------------------------------------------|
| status        | number                    | The HTTP status code (100-599)                   |
| headers       | Map[string, List[string]] | Response headers                                 |
| body          | Body                      | Response body                                    |
| matchingRules | MatchingRules             | Optional Matching rules to apply to the response |
| generators    | Generators                | Optional generators to apply to the response     |

#### Asynchronous/Messages

Asynchronous messages represent a one-way interaction between a provider and consumer. They were introduced in V3 as
Message Pacts. Each message interaction has the following additional attributes:

| Field         | Type              | Description                                                                                            |
|---------------|-------------------|--------------------------------------------------------------------------------------------------------|
| contents      | Body              | The body of the message                                                                                |
| metadata      | Map[string, JSON] | Key/value map of metadata associated with the message. Values can be any value that be stored as JSON. |
| matchingRules | MatchingRules     | Optional Matching rules to apply to the message                                                        |
| generators    | Generators        | Optional generators to apply to the message                                                            |

Example message interaction:
```json
{
  "type": "Asynchronous/Messages",
  "description": "Test Message",
  "key": "m_001",
  "metadata": {
    "contentType": "application/json",
    "destination": "a/b/c"
  },
  "providerStates": [
    {
      "name": "message exists"
    }
  ],
  "contents": {
    "contentType": "application/json",
    "encoded": false,
    "content": {
      "a": "1234-1234"
    }
  },
  "matchingRules": {
    "content": {
      "$.a": {
        "matchers": [
          {
            "match": "regex",
            "regex": "\\d+-\\d+"
          }
        ],
        "combine": "AND"
      }
    }
  },
  "generators": {
    "content": {
      "a": {
        "type": "Uuid"
      }
    }
  }
}
```

#### Synchronous/Messages

Synchronous messages represent a request/response+ interaction between a provider and consumer. They represent
non-HTTP interactions where a request message is sent to a provider, and one or more response messages are returned.
Example interactions that would use this form are gRPC and Websockets.

Each interaction has the following additional attributes:

| Field    | Type                  | Description                                          |
|----------|-----------------------|------------------------------------------------------|
| request  | MessageContents       | The the request message that is sent to the provider |
| response | List[MessageContents] | List of expected responses                           |

Example synchronous message:

```json
{
  "comments": {
    "testname": "pact::test_proto_client"
  },
  "description": "init plugin request",
  "interactionMarkup": {
    "markup": "```protobuf\nmessage InitPluginRequest {\n    string implementation = 1;\n    string version = 2;\n}\n```\n```protobuf\nmessage InitPluginResponse {\n    message .io.pact.plugin.CatalogueEntry catalogue = 1;\n}\n```\n",
    "markupType": "COMMON_MARK"
  },
  "key": "c05e8d0d3e683897",
  "pending": false,
  "pluginConfiguration": {
    "protobuf": {
      "descriptorKey": "347713ea68bb68288a09c8fd5350e928",
      "service": "PactPlugin/InitPlugin"
    }
  },
  "request": {
    "contents": {
      "content": "ChJwbHVnaW4tZHJpdmVyLXJ1c3QSBTAuMC4w",
      "contentType": "application/protobuf;message=InitPluginRequest",
      "contentTypeHint": "BINARY",
      "encoded": "base64"
    },
    "matchingRules": {
      "body": {
        "$.request.implementation": {
          "combine": "AND",
          "matchers": [
            {
              "match": "notEmpty"
            }
          ]
        },
        "$.request.version": {
          "combine": "AND",
          "matchers": [
            {
              "match": "semver"
            }
          ]
        }
      }
    }
  },
  "response": [
    {
      "contents": {
        "content": "CggIABIEdGVzdA==",
        "contentType": "application/protobuf;message=InitPluginResponse",
        "contentTypeHint": "BINARY",
        "encoded": "base64"
      },
      "matchingRules": {
        "body": {
          "$.response.catalogue": {
            "combine": "AND",
            "matchers": [
              {
                "match": "values"
              }
            ]
          },
          "$.response.catalogue.*": {
            "combine": "AND",
            "matchers": [
              {
                "match": "type"
              }
            ]
          },
          "$.response.catalogue.key": {
            "combine": "AND",
            "matchers": [
              {
                "match": "notEmpty"
              }
            ]
          },
          "$.response.catalogue.type": {
            "combine": "AND",
            "matchers": [
              {
                "match": "regex",
                "regex": "^(CONTENT_MATCHER|CONTENT_GENERATOR)$"
              }
            ]
          }
        }
      }
    }
  ],
  "type": "Synchronous/Messages"
}
```

##### MessageContents

This entity stores the contents of a message. It has the following attributes:


| Field         | Type              | Description                                                                                            |
|---------------|-------------------|--------------------------------------------------------------------------------------------------------|
| contents      | Body              | The body of the message                                                                                |
| metadata      | Map[string, JSON] | Key/value map of metadata associated with the message. Values can be any value that be stored as JSON. |
| matchingRules | MatchingRules     | Optional Matching rules to apply to the message                                                        |
| generators    | Generators        | Optional generators to apply to the message                                                            |


#### Provider states

The provider states store an indicator for the state that the provider needs to be in to be successfully
verified. They contain a `description` and a key/value map of parameters. The values in the parameters
can be any value that can be serialised to JSON.

| Field  | Type              | Description         |
|--------|-------------------|---------------------|
| name   | string            | Provider state name |
| params | Map[string, JSON] | Optional Parameters |

```json
{
  "providerStates": [
    {
      "name": "a user exists",
      "params": { 
        "name": "Testy",
        "id": 1234 
      }
    },
    {
      "name": "no administrators exist"
    }
  ]
}
```

#### Bodies

Request/Response bodies and message contents are represented with an entity with the following attributes:

| Field           | Type            | Description                                                                                                                                                 |
|-----------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| contentType     | string          | The content type of the body from the [IANA registry](https://www.iana.org/assignments/media-types/media-types.xhtml)                                       |
| encoded         | string or false | If the body has been encoded (for example, with base64), the encoding used. Otherwise false. Note for JSON stored in string form, encoded should be `JSON`. |
| contents        | string or JSON  | If encoded, must be a string value. Otherwise, can be any JSON.                                                                                             |
| contentTypeHint | string          | A hint to support how new types of content must be processed. Value must be `BINARY` or `TEXT`.                                                             |

Example:
```json
{
    "content": "Cg9wYWN0LWp2bS1kcml2ZXISBTAuMC4w",
    "contentType": "application/protobuf; message=InitPluginRequest",
    "contentTypeHint": "BINARY",
    "encoded": "base64"
}
```

#### Matching Rules

The matching rules are stored as a key/value map where the key is the category that matchers are applied to. Categories 
are string values, but the current supported ones are: body, header, path, query, metadata.

Each matching rule has the following attributes:

| Field    | Type               | Description                                                                                                   |
|----------|--------------------|---------------------------------------------------------------------------------------------------------------|
| matchers | List[MatchingRule] | List of matching rules                                                                                        |
| combine  | AND or OR          | Optional. Whether the results of applying the matching rules should be combined using an AND or OR operation. |

All matching rules must have a `type` attribute, and can have any other attributes depending on the type.

##### Single value matching rules

These are used for applying matching rules to singular values. Current only for request path and response status.

```json
{
  "path": {
    "matchers": [
      { "match": "regex", "regex": "\\w+" }
    ]
  }
}
```

##### Matching rules for collections

Stored as a key/value map, where the key is the name for headers, metadata and query parameters. For bodies, it is
a JSON Path like expression.

```json
{
  "query": {
    "Q1": {
      "matchers": [
        { "match": "regex", "regex": "\\d+" }
      ]
    }
  },
  "header": {
    "HEADERY": {
      "combine": "OR",
      "matchers": [
        {"match": "include", "value": "ValueA"},
        {"match": "include", "value": "ValueB"}
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

##### Supported matching rules

| matcher       | Spec Version | example configuration                                                                                      | description                                                                                                                                                                                                                            |
|---------------|--------------|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Equality      | V1           | `{ "match": "equality" }`                                                                                  | This is the default matcher, and relies on the equals operator                                                                                                                                                                         |
| Regex         | V2           | `{ "match": "regex", "regex": "\\d+" }`                                                                    | This executes a regular expression match against the string representation of a values.                                                                                                                                                |
| Type          | V2           | `{ "match": "type" }`                                                                                      | This executes a type based match against the values, that is, they are equal if they are the same type.                                                                                                                                |
| MinType       | V2           | `{ "match": "type", "min": 2 }`                                                                            | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the minimum.             |
| MaxType       | V2           | `{ "match": "type", "max": 10 }`                                                                           | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the maximum.             |
| MinMaxType    | V2           | `{ "match": "type", "max": 10, "min": 2 }`                                                                 | This executes a type based match against the values, that is, they are equal if they are the same type. In addition, if the values represent a collection, the length of the actual value is compared against the minimum and maximum. |
| Include       | V3           | `{ "match": "include", "value": "substr" }`                                                                | This checks if the string representation of a value contains the substring.                                                                                                                                                            |
| Integer       | V3           | `{ "match": "integer" }`                                                                                   | This checks if the type of the value is an integer.                                                                                                                                                                                    |
| Decimal       | V3           | `{ "match": "decimal" }`                                                                                   | This checks if the type of the value is a number with decimal places.                                                                                                                                                                  |
| Number        | V3           | `{ "match": "number" }`                                                                                    | This checks if the type of the value is a number.                                                                                                                                                                                      |
| Timestamp     | V3           | `{ "match": "datetime", "format": "yyyy-MM-dd HH:ss:mm" }`                                                 | Matches the string representation of a value against the datetime format                                                                                                                                                               |
| Time          | V3           | `{ "match": "time", "format": "HH:ss:mm" }`                                                                | Matches the string representation of a value against the time format                                                                                                                                                                   |
| Date          | V3           | `{ "match": "date", "format": "yyyy-MM-dd" }`                                                              | Matches the string representation of a value against the date format                                                                                                                                                                   |
| Null          | V3           | `{ "match": "null" }`                                                                                      | Match if the value is a null value (this is content specific, for JSON will match a JSON null)                                                                                                                                         |
| Boolean       | V3           | `{ "match": "boolean" }`                                                                                   | Match if the value is a boolean value (booleans and the string values `true` and `false`)                                                                                                                                              |
| ContentType   | V3           | `{ "match": "contentType", "value": "image/jpeg" }`                                                        | Match binary data by its content type (magic file check)                                                                                                                                                                               |
| Values        | V3           | `{ "match": "values" }`                                                                                    | Match the values in a map, ignoring the keys                                                                                                                                                                                           |
| ArrayContains | V4           | `{ "match": "arrayContains", "variants": [...] }`                                                          | Checks if all the variants are present in an array.                                                                                                                                                                                    |
| StatusCode    | V4           | `{ "match": "statusCode", "status": "success" }`                                                           | Matches the response status code.                                                                                                                                                                                                      |
| NotEmpty      | V4           | `{ "match": "notEmpty" }`                                                                                  | Value must be present and not empty (not null or the empty string)                                                                                                                                                                     |
| Semver        | V4           | `{ "match": "semver" }`                                                                                    | Value must be valid based on the semver specification                                                                                                                                                                                  |
| EachKey       | V4           | `{ "match": "eachKey", "rules": [{"match": "regex", "regex": "\\$(\\.\\w+)+"}], "value": "$.test.one" }`   | Allows defining matching rules to apply to the keys in a map                                                                                                                                                                           |
| EachValue     | V4           | `{ "match": "eachValue", "rules": [{"match": "regex", "regex": "\\$(\\.\\w+)+"}], "value": "$.test.one" }` | Allows defining matching rules to apply to the values in a collection. For maps, delgates to the Values matcher.                                                                                                                       |

#### Generators

The generators, just like matching rules, are stored as a key/value map where the key is the category that generators are applied to. Categories 
are string values, but the current supported ones are: body, header, path, query, metadata, status.

| Field      | Type                   | Description                     |
|------------|------------------------|---------------------------------|
| generators | Map[string, Generator] | Map of categories to generators |

All generators must have a `type` attribute, and can have any other attributes depending on the type.

##### Single value generator categories

These are used for applying generators to singular values. Current only request paths and response statuses.

```json
{
  "generators": {
    "status": {"max": 299, "min": 200, "type": "RandomInt"},
    "path": {"regex": "\\d+", "type": "Regex"}
  }
}
```

##### Generators for collections

Stored as a key/value map, where the key is the name for headers, metadata and query parameters. For bodies, it is
a JSON Path like expression.

```json
{
  "generators": {
    "body": {
        "$.1": {"digits": 4, "type": "RandomDecimal"},
        "$.2": {"digits": 4, "type": "RandomDecimal"}
    },
    "header": {
        "A": {"digits": 4, "type": "RandomDecimal"},
        "B": {"digits": 4, "type": "RandomDecimal"}
    },
    "query": {
        "a": {"digits": 4, "type": "RandomDecimal"},
        "b": {"digits": 4, "type": "RandomDecimal"}
    }
  }
}
```

##### Supported generators

| matcher                | Spec Version | example configuration                                                                                        | description                                                                                                                                                                                                                                                                                             |
|------------------------|--------------|--------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| RandomInt              | V3           | `{"max": 299, "min": 200, "type": "RandomInt"}`                                                              | Generates a random integer between the min and max values (inclusive)                                                                                                                                                                                                                                   |
| RandomDecimal          | V3           | `{"digits": 8, "type": "RandomDecimal"}`                                                                     | Generates a random big decimal value with the provided number of digits                                                                                                                                                                                                                                 |
| RandomHexadecimal      | V3           | `{"digits": 8, "type": "RandomHexadecimal"}`                                                                 | Generates a random hexadecimal value of the given number of digits                                                                                                                                                                                                                                      |
| RandomString           | V3           | `{"size": 20, "type": "RandomString"}`                                                                       | Generates a random alphanumeric string of the provided length                                                                                                                                                                                                                                           |
| Regex                  | V3           | `{"regex": "\\d+", "type": "Regex"}`                                                                         | Generates a random string from the provided regular expression                                                                                                                                                                                                                                          |
| Uuid                   | V3/V4        | `{"format": "simple", "type": "Uuid"}`                                                                       | Generates a random UUID. V4 supports specifying the format: simple (e.g 936DA01f9abd4d9d80c702af85c822a8), lower-case-hyphenated (e.g 936da01f-9abd-4d9d-80c7-02af85c822a8), upper-case-hyphenated (e.g 936DA01F-9ABD-4D9D-80C7-02AF85C822A8), URN (e.g. urn:uuid:936da01f-9abd-4d9d-80c7-02af85c822a8) |
| Date                   | V3           | `{"format": "yyyy-MM-dd", "type": "Date"}`                                                                   | Generates a date value for the provided format. If no format is provided, ISO date format is used. If an expression is given, it will be evaluated to generate the date, otherwise 'today' will be used                                                                                                 |
| Time                   | V3           | `{"format": "HH:mm::ss", "type": "Date"}`                                                                    | Generates a time value for the provided format. If no format is provided, ISO time format is used. If an expression is given, it will be evaluated to generate the time, otherwise 'now' will be used                                                                                                   |
| DateTime               | V3           | `{"format": "YYYY-mm-DD'T'HH:mm:ss", "type": "DateTime"}`                                                    | Generates a datetime value for the provided format. If no format is provided, ISO format is used. If an expression is given, it will be evaluated to generate the datetime, otherwise 'now' will be used                                                                                                |
| RandomBoolean          | V3           | `{"type": "RandomBoolean"}`                                                                                  | Generates a random boolean value                                                                                                                                                                                                                                                                        |
| ProviderState          | V4           | `{"expression": "/api/user/${id}", type": "ProviderState"}`                                         | Generates a value that is looked up from the provider state context using the given expression                                                                                                                                                                                                          |
| MockServerURL          | V4           | `{"regex": ".*\\/(orders\\/\\d+)$", "example": "http://localhost:1234/orders/5678", type": "MockServerURL"}` | Generates a URL with the mock server as the base URL.                                                                                                                                                                                                                                                   |

#### Comments

The comment attribute provides a key-value map (Map[string, JSON]) to store associated comments in the Pact file.
An example use of this is to store the name of the test that generated the pact. See issue #45 for more details.

| Key      | Description                                                      |
|----------|------------------------------------------------------------------|
| testname | Stores the name of the test that ran and generated the Pact file |
| text     | List of free-form text comments to display                       |

Example:

```json
{
    "testname": "runTest(au.com.dius.pact.consumer.junit.v4.V4HttpPactTest)",
    "text": [
      "This is a comment",
      "Another comment",
      "This is also a comment"
    ]
}
```

#### Interaction Markup

To support data formats that may be added by third-party plugins, interaction markup entity allows markup to be
provided to support rendering the format in UIs. HTML and Common Mark are supported. 

It has the following attributes:

| Field      | Type   | Description                                         |
|------------|--------|-----------------------------------------------------|
| markup     | string | The markup for the interaction in string format     |
| markupType | string | The type of markup. Must be `COMMON_MARK` or `HTML` |

Example:
```json
{
  "markup": "```protobuf\nmessage InitPluginRequest {\n    string implementation = 1;\n    string version = 2;\n}\n```\n```protobuf\nmessage InitPluginResponse {\n    message .io.pact.plugin.CatalogueEntry catalogue = 1;\n}\n```\n",
  "markupType": "COMMON_MARK"
}
```

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
