<pre>
  WIP: WIP-ASDPC-NUMBER-SEPARATORS
  Layer: Consensus (hard fork)
  Title: Improved processing of numbers in oracle queries
  Authors: Adán SDPC <adan@witnet.foundation>
  Discussions-To: `#dev-lounge` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2022-07-27
  License: BSD-2-Clause
</pre>


## Abstract

This introduces additional rules when processing in Witnet's data retrieval and aggregation engine (RAD), such that
APIs that provide numbers formatted in a non-conventional manner can also be used as data sources.

## Motivation and rationale

Witnet is designed as a generalist oracle: there is no need for the nodes to select and publish specific datasets
that they want to offer to smart contracts. On the contrary, it is rather on the data consumers (the smart contract
developers) to explicitly select multiple data sources to be queried over HTTPS or orther protocols, define how to
process the responses, and how to filter and aggregate the output of those multiple sources. In the Witnet protocol,
we often call that specification _data request_, or more recently, _oracle query_. 

This generalist approach has been proven more than effective to create decentralization at the data source level, as
it enables the user to pick virtually any publicly accessible APIs as their data sources, while mitigating trust in any 
single of those by means of basic filtering and aggregation techniques.

This approach comes however with a few challenges, as public APIs come in all sizes and colors, and each of them will
format data differently. We are not only speaking of the data structure or _schema_, but also how different data
types are encoded.

One specific data type that APIs often tend to encode in non-conventional ways are numbers. Even though there are
international standards on how to represent numbers, there are also alternative representations (often regional) that
can be easily found when trying to collect a broad number of data sources for composing a solid oracle query.

Namely, the main differences between those number representation systems lie in the use of different separators for
telling apart the integer and the decimal part of a number; as well as those used for marking thousands, millions,
billions, etc.

This is a sample of how the same number may be represented by different APIs found in the wild:

```
1234.567
1,234.567
1234,567
1.234,567
1 234.567
1 234,567
```

This improvement proposal gives the "standard" status to the first format above (`.` as the decimal separator, and no
separator at all for thousands, millions, etc.), and requires all Witnet protocol implementations to assume that, unless
otherwise specified by the oracle query, all numbers come in that format. May an API use a non-conventional formatting,
the oracle query will need to state what are the separators being used, by providing those characters as additional
arguments to any number processing RAD operators.

## Specification

### Definitions

- For the scope of this proposal, the _number operators_ are `Array::get_float`, `Array::get_integer`, `Map::get_float`,
`Map::get_integer`, `String::to_float`, and `String::to_integer`.

- A _decimal separator_ is the character or sequence of characters that mark the separation between the integer part and
the decimal part of a number.

- A _thousands separator_ is the character or sequence of characters that mark thousands, millions, billions, etc. in 
big numbers.

### Additional arguments in number operators

**(1)** All number operators MUST accept two more arguments than they used to, being the first one of those reserved
for stating the thousands separator, and the second one for the decimal separator.

**(2)** All thousands separators and decimal separators provided MUST be valid UTF-8 characters or sequences of UTF-8
characters, encoded into a `String`.

**(3)** May the thousands separator argument not be provided, the thousands separator MUST default to be the comma
symbol `,` (Unicode `U+002C`, UTF-8 `0x2c`).
 
**(4)** May the decimal separator argument not be provided, the decimal separator MUST default to be the full stop
symbol `.` (Unicode `U+002E`, UTF-8 `0x2e`).

**(5)** May the decimal separator argument be provided, the thousands separator argument MUST be provided too.

### Pre-processing of separators in number operators

**(6)** All number operators MUST pre-process numbers in such a way that any occurrence of the user-provided thousands 
separator or the default thousands separator as defined in (3) are removed from the number (or equivalently, replaced
by an empty string).

**(7)** All number operators MUST pre-process numbers in such a way that any occurrence of the user-provided decimal
separator are replaced by the default decimal separator as defined in (4).

**(8)** Replacement of decimal separators do not affect processing of integer numbers, and hence the decimal separator 
argument can be elided for number operators that output integer numbers.

## Backwards compatibility

- Implementer: a client that implements or adopts the protocol improvements proposed in this document.
- Non-implementer: a client that fails to or refuses to implement the protocol improvements proposed in this document.


#### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions considered valid by former versions of the protocol MUST continue to be considered valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with
those coming from implementers, that is, the new rules.
- Implementers MUST apply the proposed logic when resolving oracle queries.

As a result:

- Commitments and reveals signed by non-implementers MAY differ from those signed by implementers.
- Oracle queries MAY resolve to `InsufficientConsensus` given a witnessing committee formed by a mixed sample of
implementers and non-implementers.

Due to this last points, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be
proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.


### Libraries, clients, and interfaces

The protocol improvements proposed herein will affect any library or client implementing any functionality related to
the RAD engine. For example, this is the case of the [Sheikah][sheikah] data request editor and the [witnet-radon-js]
and [witnet-requests-js] Javascript libraries.

## Reference Implementation

A reference implementation for the proposed protocol improvement can be found in the
[pull request #2235](https://github.com/witnet/witnet-rust/pull/2235) of the [witnet-rust] repository.

## Adoption Plan

An activation date and [TAPI] signaling bit will be opportunely proposed upon moving this draft to the Proposed stage.

## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community.

Special thanks to @drcpu-github for contributing to the reference implementation in Rust.

[sheikah]: https://github.com/witnet/sheikah
[witnet-radon-js]: https://github.com/witnet/witnet-radon-js
[witnet-requests-js]: https://github.com/witnet/witnet-requests-js
[witnet-rust]: https://github.com/witnet/witnet-rust/
[TAPI]: wip-0014.md
