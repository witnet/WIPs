<pre>
  WIP: WIP-encode-error
  Layer: Consensus (hard fork)
  Title: Introduce new EncodeReveal RADON error
  Authors: Tomasz Polaczyk <tomasz@witnet.foundation>
  Discussions-To: `#dev-general` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2023-01-12
  License: BSD-2-Clause
</pre>


## Abstract

A new `EncodeReveal` RADON error is used as the result of a RADON aggregation script when the result cannot be encoded to CBOR.

## Motivation and rationale

There are some values that can be represented as RADON types, but cannot be represented using CBOR. For example, see this witnet request:

```js
const big_integer = new Witnet.Source("http://example.org/array_with_1_element.json")
    .parseJSONArray()
    .getInteger(0)
    .multiply(1000000000000000)
    .multiply(100000)
```

Where the source response is a JSON array with value `[1]`.

The result of that retrieval script will be a `RadonInteger` greater than the maximum allowed by CBOR. This will not immediately result in an error, because the result of a retrieval script is not directly encoded to CBOR. However, if the aggregation script does not modify that value, by using a simple reducer like Mode, then the result of the aggregation script will be a `RadonInteger` greater than the maximum allowed by CBOR, and that will result in an encode error when trying to create the reveal transaction.

Before this proposal, the encode error was not reported as the result of the data request. If a node failed to create the reveal transaction, it then also gave up on creating the commit transaction, and participating in the data request. This would most probably make the data request result in an `InsufficientCommits` error, because no nodes were able to participate in the resolution of that request.

After this proposal, that same scenario will result in a EncodeReveal error as the result of the data request, which will help to detect issues related to encoding RADON to CBOR. This new RADON error is not limited to the case of integers that are greater than the maximum allowed by CBOR, it also applies to any other possible encode errors. Currently there are no other known issues that could result in this error, but that can change as the RADON language is expanded with new operators.

In the case of tally transactions, if the result of the tally script cannot be encoded, nodes currently report `RadError::Unknown` as the result of the data request. To ensure backwards compatibility, the `EncodeReveal` error will only be used to handle the encoding of the result of the aggregation script. The encoding of the tally script will keep its old behavior.

## Specification

Starting from and including the activation epoch:

**(1)**: When the result of the aggregation stage of a data request cannot be encoded as a CBOR value, the result will be the new `EncodeReveal` RADON error, with error code `0x61` and no arguments.


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

This change only affects the behavior of some data requests that previously resolved to InsufficientCommits, but after this change will resolve to EncodeReveal error.

Libraries such as the witnet toolkit or the witnet wallet will not see this error because they never encode the result to CBOR. However it would be a nice improvement in user experience to check that the result can actually be encoded, as otherwise the request will not work as expected when deployed to the witnet network.

The introduction of a new RADON error means that the solidity libraries need to be updated to handle that error if we want users to see a nice error message. Not updating will result in a generic `Unknown error (0x61)`, but will not break any code, so it is optional.

## Reference Implementation

A reference implementation for the proposed protocol improvement can be found in the [encode-encode-error branch](https://github.com/tmpolaczyk/witnet-rust/tree/encode-encode-error) of the [witnet-rust] repository.


## Adoption Plan

An adoption plan will be proposed upon moving this document from the _Draft_ stage to the _Proposed_ stage.


## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community.

[witnet-rust]: https://github.com/witnet/witnet-rust/
