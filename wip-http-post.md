<pre>
  WIP: WIP-XXXX
  Layer: Consensus (hard fork)
  Title: Support HTTP-POST in RADON
  Authors: Tomasz Polaczyk <tomasz@witnet.foundation>
  Discussions-To: `#dev-general` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2021-12-02
  License: BSD-2-Clause
</pre>


## Abstract

RADON supports HTTP retrieval sources, but only using the HTTP verb GET. Some APIs require the use of the HTTP verb POST. This proposal specifies support for HTTP POST requests.

## Motivation and rationale

While HTTP-GET requests can cover the majority of use cases, there are some APIs that require the use of POST requests. The most common example is the case of GraphQL APIs, which use a POST request with a JSON body to specify what kind of data will be retrieved.

## Specification

At the protocol level, these are the new fields in the protobuf schema:

```diff
 enum RADType {
     Unknown = 0;
     HttpGet = 1;
     Rng = 2;
+    HttpPost = 3;
 }
 message RADRetrieve {
     RADType kind = 1;
     string url = 2;
     // TODO: RADScript should maybe be a type?
     bytes script = 3;
+    // Body of HTTP-POST request
+    bytes body = 4;
+    // Extra headers for HTTP-GET and HTTP-POST requests
+    repeated StringPair headers = 5;
 }
```

#### HTTP-POST

This retrieval type is similar to HTTP-GET, except:

* Using POST instead of GET.
* Allowing an additional `body` parameter, which is an array of bytes that will be sent to the HTTP server.

#### New `body` field

When the `RADType` is HTTP-GET or RNG, the `body` field must be empty.

When the `RADType` is HTTP-POST, the `body` field may contain an arbitrary sequence of bytes.

The content of this field will be sent to the HTTP server as part of the POST request.

#### New `headers` field

When the `RADType` is RNG, the `headers` field must be empty.

When the `RADType` is HTTP-GET or HTTP-POST, the `headers` field may contain a list of `(key, value)` pairs, using the `StringPair` helper message:

```
message StringPair {
    string left = 1;
    string right = 2;
}
```

These pairs will be used as additional headers for the HTTP request. At the protocol level, they can be any string, but the RAD engine will validate them according to the HTTP specification, and resolve the retrieval source to a `RadonError` in case of error.

Since HTTP-GET requests follow a very similar structure to HTTP-POST requests, the addition of the `headers` field will also be implemented in the HTTP-GET handler. The semantics are exactly the same as in HTTP-POST requests.

#### Updated transaction weights

The new fields of the `RADRetrieve` message must be taken into account when calculating the data request transaction weight.

Old weight:

```
RADRetrieve weight = (script bytes) + (url bytes) + (1)
```

New weight:

```
RADRetrieve weight = (script bytes) + (url bytes) + (1) + (body bytes) + (headers_weight)
headers_weight = for each header: (key bytes) + (value bytes) + overhead
overhead = 6
```

The overhead refers to the serialization overhead of `StringPair`, in an attempt to make the weight match the serialized size as close as possible.

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

A reference implementation for the proposed protocol improvement can be found in the [http-post](https://github.com/tmpolaczyk/witnet-rust/commits/http-post) branch of the [witnet-rust] repository.


## Adoption Plan

An adoption plan will be proposed upon moving this document from the _Draft_ stage to the _Proposed_ stage.


## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community.

[witnet-rust]: https://github.com/witnet/witnet-rust/
