<pre>
  WIP: WIP-0029
  Layer: Consensus (hard fork)
  Title: Support HTTP-HEAD in RADON
  Authors: Guillermo Díaz <guillermo@otherplane.com>
  Discussions-To: ..tbd..
  Status: Draft
  Type: Standards Track
  Created: 2024-02-22
  License: BSD-2-Clause
</pre>


## Abstract

Data Requests in the Witnet blockchain currently support the aggregation of data extracted from any combination of  HTTP-GET and HTTP-POST retrievals. This proposal specifies the support of HTTP-HEAD data retrievals, enabling data requests to extract metadata about pre-existing content on the Internet. 

## Motivation and rationale

While HTTP-GET and HTTP-POST data retrievals may cover the majority of use cases, other potential applications rather need to retrieve metadata from allegedly existing resource contents on the Internet, instead of relyng on calls to REST/API endpoints.  

For instance, metadata like the existence, age, size, hash or the content type of some given URL, can be extracted by a single HTTP-HEAD retrieval, without actually forcing witnessing nodes to download the whole content that is being referred. 

## Specification

At the protocol level, these are the new fields in the protobuf schema:

```diff
 enum RADType {
     Unknown = 0;
     HttpGet = 1;
     Rng = 2;
     HttpPost = 3;
+    HttpHead = 4;  
 }
 ```

### HTTP-HEAD

#### Optional `headers` field

Just like HTTP-GET and HTTP-POST retrievals, this new retrieval type will also accept including request headers.

The `headers` field may contain a list of `(key, value)` pairs, using the `StringPair` helper message:

```
message StringPair {
    string left = 1;
    string right = 2;
}
```

These pairs will be used as additional headers for the underlying HTTP request. At the protocol level, they can be any string, but the RAD engine will validate them according to the HTTP specification, and resolve the retrieval source to a `RadonError` in case of error.


#### Transaction weights

The weight of a HTTP-HEAD retrievals are calculated just the same as HTTP-GET ones:

```
RADRetrieve weight = (script bytes) + (url bytes) + (1) + (body bytes) + (headers_weight)
headers_weight = for each header: (key bytes) + (value bytes) + overhead
overhead = 6
```

The overhead refers to the serialization overhead of `StringPair`, in an attempt to make the weight match the serialized size as close as possible.

#### HTTP-HEAD response format

Witnet nodes performing HTTP-HEAD retrieval will pass as input value to the corresponding Radon script a stringified JSON object containing one entry per response header as returned from the remote host.  

For instance, whereas the HTTP-HEAD request to https://www.wikipedia.org/static/favicon/wikipedia.ico returns these lines:
```
HTTP/1.1 200 OK
date: Thu, 22 Feb 2024 16:23:24 GMT
server: mw-web.eqiad.main-f5dfc99f8-7mnkk
last-modified: Wed, 29 Nov 2023 14:11:57 GMT
cache-control: max-age=31536000
expires: Fri, 21 Feb 2025 16:23:24 GMT
access-control-allow-origin: *
content-type: image/vnd.microsoft.icon
etag: W/"aae-60b4b1e2d4540"
vary: Accept-Encoding
age: 4894
x-cache: cp6012 miss, cp6009 hit/151018
x-cache-status: hit-front
server-timing: cache;desc="hit-front", host;desc="cp6009"
strict-transport-security: max-age=106384710; includeSubDomains; preload
x-client-ip: 90.167.7.109
accept-ranges: bytes
content-length: 2734
```

The `RadonString` value passed as input to the Radon script would rather be encoded as:

```
{
    "date": "Thu, 22 Feb 2024 16:23:24 GMT",
    "server": "mw-web.eqiad.main-f5dfc99f8-7mnkk",
    "last-modified": "Wed, 29 Nov 2023 14:11:57 GMT",
    "cache-control": "max-age=31536000",
    "expires": "Fri, 21 Feb 2025 16:23:24 GMT",
    "access-control-allow-origin": "*",
    "content-type": "image/vnd.microsoft.icon",
    "etag": "W/\"aae-60b4b1e2d4540\"",
    "content-encoding": "gzip",
    "vary": "Accept-Encoding",
    "age": "4802",
    "x-cache": "cp6012 miss, cp6009 hit/148083",
    "x-cache-status": "hit-front",
    "server-timing": "cache;desc=\"hit-front\", host;desc=\"cp6009\"",
    "strict-transport-security": "max-age=106384710; includeSubDomains; preload",
    "x-client-ip": "90.167.7.109",
    "accept-ranges": "bytes",
    "content-length": "1035"
}
```

## Backwards compatibility

- Implementer: a client that implements or adopts the protocol improvements proposed in this document.
- Non-implementer: a client that fails to or refuses to implement the protocol improvements proposed in this document.


#### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions that were considered valid by former versions of the protocol MUST continue to be considered valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with those coming from implementers, that is, the new rules.
- Implementers MUST apply the proposed logic when evaluating transactions or computing their own data request eligibility.

As a result:

- Non-implementers MAY NOT get their blocks and transactions accepted by implementers.
- Implementers MAY NOT get their valid blocks accepted by non-implementers.
- Non-implementers MAY consolidate different blocks and superblocks than implementers.

Due to this last point, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.


### Libraries, clients, and interfaces

From the perspective of a data request creator this is a backwards compatible change: all the data requests that were valid before this change will still be valid after this change.

All the libraries that allow creation or visualization of data requests will need to be updated in order to support the new functionality.

## Reference Implementation

A reference implementation for the proposed protocol improvement can be found in the [http-head](https://github.com/witnet/witnet-rust/pull/2402) branch of the [witnet-rust] repository.


## Adoption Plan

TBD...

## Acknowledgements

This proposal has been cooperatively discussed and devised by individuals from the Witnet development community.

[witnet-rust]: https://github.com/witnet/witnet-rust/
