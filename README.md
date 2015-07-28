# Introduction

["Pact"](https://github.com/realestate-com-au/pact) is an implementation of "consumer driven contract" testing that allows mocking of responses in the consumer codebase, and verification of the interactions in the provider codebase. The initial implementation was written in Ruby for Rack apps, however a consumer and provider may be implemented in different programming languages, so the "mocking" and the "verifying" steps would be best supported by libraries in their respective project's native languages. Given that the pact file is written in JSON, it should be straightforward to implement a pact library in any language, however, to get the best experience and most reliability of out mixing pact libraries, the matching logic for the requests and responses needs to be identical. There is little confidence to be gained in having your pacts "pass" if the logic used to verify a "pass" is inconsistent between implementations.

To support consistency of matching logic, this specification has been developed as a benchmark that all pact libraries can check themselves against if they want to ensure consistency with other pact libraries.

### Pact Specificaton Philosophy

The Pact matching rules follow [Postel's Law][postel].

* Be as strict as we reasonably can with what we send out (requests). We should know and be able to control exactly what a consumer is sending out. Information should not be allowed to "leak" silently.
* Be as loose as we reasonably can with what we accept (responses). A provider should be able to send extra information that this particular consumer does not care about, without breaking this consumer.
* When writing the matching rules, err on the side of being more strict now, because it will break fewer things to be looser later, than to get stricter later.

Note: One implications of this philosophy is that you cannot verify, using pact, that a key or a header will _not_ be present in a response. You can only verify what _is_.

### Goals for implementing a pact library
* The DSLs should be as similar as possible, within the idioms of the language, to the original Ruby wording to help create a consistent Pact experience, and minimise overhead when switching between languages.
* The matching logic for the requests and responses should be the same so that we can have the same amount of confidence that the integrations will work in real life not matter what language the system was written and tested in.


# Index

* [Version 1](https://github.com/bethesque/pact_specification/tree/version-1) - A spec that describes the existing matching in the ruby implementation, and provides test cases for implementations in other languages to ensure that matching is consistent.

* [Version 1.1](https://github.com/bethesque/pact_specification/tree/version-1.1) - Updated specification shared between ruby, JVM and .Net versions.

* [Version 2](https://github.com/bethesque/pact_specification/tree/version-2) - Introduces non-language specific regular expression and type matching.

* [Version 3](https://github.com/bethesque/pact_specification/tree/version-3) - WIP - Introduces pact format for message queues and corrects some issues with V2.

[postel]: http://en.wikipedia.org/wiki/Jon_Postel#Postel.27s_Law
