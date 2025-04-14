<pre>
  WIP: WIP-0029
  Layer: Consensus (hard fork)
  Title: Support of binary sources
  Authors: Guillermo Díaz <guillermo@otherplane.com>
  Discussions-To: https://github.com/witnet/witnet-rust/discussions/2433
  Status: Draft
  Type: Standards Track
  Created: 2024-02-23
  License: BSD-2-Clause
</pre>


## Abstract

At the moment, Radon scripts are only capable of processing stringified responses from HTTP-GET and HTTP-POST retrievals. This proposal specifies the support of binary buffers as returned from HTTP-GET or HTTP-POST retrievals.

## Motivation and rationale

Supporting binary sources enables Data Requests to process the whole binary content, or just parts of it, of any public resource or digital asset published on the Internet. An example of a possible application would be leveraging the witnessing capability of the Witnet blockchain to certify not only the existence of some public digital asset at certain moment of time, but also capture a cryptographic proof (e.g. SHA-256 hash) of its actual content. 

## Specification

Upon execution of a HTTP-GET or HTTP-POST retrieval, the first operator of the corresponding Radon script needs to be peeked in order to determine whether to deserialize the response from the HTTP source as a string or as a buffer stream. As for the scope of this proposal, both RadonString and RadonBytes operators would be acceptable as first element in Radon scripts. If a Radon operator of any other kind is found as first element in the retrieval's Radon script, a `RadError::InvalidScript` should be triggered before even attempting the corresponding HTTP-GET or HTTP-POST request. 

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

A reference implementation for the proposed protocol improvement can be found in the [feat/binary-sources](https://github.com/witnet/witnet-rust/pull/2406) branch of the [witnet-rust] repository.


## Adoption Plan

TBD...

## Acknowledgements

This proposal has been cooperatively discussed and devised by individuals from the Witnet development community.

[witnet-rust]: https://github.com/witnet/witnet-rust/
