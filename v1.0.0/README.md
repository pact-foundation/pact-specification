### Version 1.0.0

#### Request matching

1. Request method

Exact string match, case sensitive (expects lower case). Open to the idea that this could be case insensitive.

2. Request path

Exact string match, case sensitive, as paths are generally case sensitive. Trailing slashes should not be ignored, as they could potentially have significance.

3. Request headers

Exact string match for expected header names and values. Allow unexpected headers to be sent out, as frameworks and network utilities are likely to set their own headers (eg. User-Agent), and it would increase the maintenance burden to have to track all of those.

4. Request body

* Do not allow unexpected keys to be sent in the body. See "Pact Specificaton Philosophy" in main README.
* Do not allow unexpected items in an array. Most parsing code will do a "for each" on an array, and if we expect one item, but two go out, we have "leaked" information.

#### Response matching

1. Response status

Exact integer match.

2. Response headers

Exact string match for expected header names and values. Allow unexpected headers to be sent back, as in reality, as extra headers will be added by network utilities and server frameworks.

3. Response body

* Allow unexpected keys to be sent back in the body. See "Pact Specificaton Philosophy" in main README.
* Do not allow unexpected items in an array. Most parsing code will do a "for each" on an array, and if we expect one item, but receive two, we might not have exercised the correct code to handle that second item in our consumer tests.
