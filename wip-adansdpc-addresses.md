    WIP: WIP-adansdpc-addresses
    Title: Value transfer outputs and addresses
    Author: Adán SDPC <adan@witnet.foundation>
    Comments-Summary: No comments yet.
    Comments-URI: https://github.com/witnet/wips/wiki/Comments:WIP-adansdpc-addresses
    Status: Draft
    Type: Process
    Created: 2018-05-08
    License: BSD-2-Clause

## Abstract

This document proposes a standard for value transfer outputs and transactions, and a checksummed base32 format ("Bech32") to represent those outputs as exchangeable and human friendly addresses.

The formats set forth hereinafter are inspired on [BIP-0141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) and [BIP-0173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki), two documents introducing significant improvements over the Bitcoin protocol that were long needed for historical reasons.

## Copyright

This WIP is heavily based on Bitcoin's [BIP-0141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki) and [BIP-0173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki).
Therefore it inherits their original licensing: the 2-clause BSD license.

## Motivation

The motivation to adopt this kind of output and transaction format (often referred to as _segregated witness_) is to guarantee maximum extensibility of output formats and to avoid transaction malleability from the early days of the Witnet network.

This is efficiently achieved by keeping signatures out of the transaction structure committed to the transaction merkle tree.
This is possible because the entirety of the transaction's effects are determined by output consumption (spends) and new output creation.
Signatures are only required to validate the blockchain state, not to determine it.

This separation between the semantic kernel of the transactions and the signature data gives a number of advantages:

1. __Unintentional malleability becomes impossible__. Since signature data is not committed to the transaction hash, changes to how the transaction was signed are no longer relevant to transaction identification.
2. __Transmission of signature data becomes optional__. It is needed only if a peer is trying to validate a transaction instead of just checking its existence. This reduces the size of SPV proofs and potentially improves the privacy of SPV clients as they can download more transactions using the same bandwidth.
3. __Some constraints can be bypassed with a soft fork__. By including a special parameter whose value can be given different meanings in the future, we can signal usage of different output formats nested inside this one, which in example opens the door to supporting multiple output contract languages.

## Specification

### Output format

Every output in a Witnet transaction must abide by the following format:

| Size    | Description |
|---------|-------------|
| [flex-3](https://github.com/aesedepece/WIPs/blob/wip-chain/wip-adansdpc-chain.md#flex-3)   | Relative value to be assigned to this output | 
| [flex-b1](https://github.com/aesedepece/WIPs/blob/wip-chain/wip-adansdpc-chain.md#flex-b1) | Contract type bits |
| any     | Output contract |

#### Contract type bits

Contract type bits are integers ranging from 0 to 127, codified using the [flex-b1](https://github.com/aesedepece/WIPs/blob/wip-chain/wip-adansdpc-chain.md#flex-b1) encoding, which saves half a byte when using output contracts of type `0` to `7`.
They indicate which contract type is being used. That is, how the output contract must be interpreted.

##### Contract type bits registry

| Bits    | Value | Contract type | Standardized by |
|---------|-------|---------------|-----------------|
| `b0000` | 0     | Equivalent to Bitcoin's P2WPKH / P2WSH (native SegWit)  | This document |

Future WIPs can propose addition of new entries to this registry.

#### Output contract

Output contracts contain programming code stating the spending conditions for the output their are included into.

The term contract is used here to mean that the system commits to allow spending of the coins enclosed in the output as long as the conditions stated by itself are satisfied by the spending transaction.

Output contracts are much equivalent to Bitcoin's _witness programs_, but we have stayed away from using the _witness_ term to avoid confusion with its own meaning in the Witnet ecosystem, where it means a type of network node.

For the sake of simplicity, usage of arbitrary (non-standard) scripts is not directly enabled by any of the output contract types specified in this document.
If required, an output contract with type `0` can be constructed to make the output be spendable by a transaction providing an arbitrary script. Such outputs are commonly known as _pay-to-script-hash_ (P2SH). 

### Output contract type 0

If the contract type is `0`, and the output contract is 20 bytes:
* It is interpreted as a _pay-to-public-key-hash_ (P2PKH) output contract.
* Exactly two spending arguments must be provided. The first one a signature, and the second one a public key.
* The first 20 bytes of the SHA256 hash of the public key must match the 20-byte output contract.
* After normal script evaluation, the signature is verified against the public key with CHECKSIG operation. The verification must result with a single TRUE on the stack.

If the contract type is `0`, and the output contract is 32 bytes:
* It is interpreted as a _pay-to-script-hash_ (P2SH) output contract.
* The output arguments must be a series of stack items followed by a serialized script.
* The script is hashed using SHA256 and matched against the 32-byte output contract.
* The script is then deserialized and executed taking the remaining stack items from the output arguments as arguments for the script itself. 
* The script must not fail, and result in exactly a single TRUE on the stack.

If the contract type is `0`, and the output contract is neither 20 nor 32 bytes, the script must fail.

If the contract type is not `0`, any logic predating this document remains unchanged.

### Examples

#### P2PKH

The following example is a type `0` _pay-to-public-key-hash_ (P2PKH):

    full output:        1 0 <20-byte-key-hash>
    contract type:      0
    contract:           <20-byte-key-hash>
    spending arguments: <signature> <pubKey>

The `0` in the full output indicates that the output contract must be interpreted as a _type 0_ contract.
The length of the output contract indicates that it is a P2PKH subtype.
The spending arguments must consist of exactly 2 items.
The first 20 bytes of the SHA256 hash of the `pubKey` in the spending arguments must match the output contract.

The signature is verified as:

    <signature> <pubKey> CHECKSIG

#### P2SH

The following example is a theoretical 1-of-2 multi-signature version `0` _pay-to-script-hash_:

    full output:        1 0 <32-byte-script-hash>
    contract type:      0
    contract:           <32-byte-script-hash>
    spending arguments: 0 <signature1> <1 <pubKey1> <pubKey2> 2 CHECKMULTISIG>

The `0` in the full output indicates that the output contract must be interpreted as a _type 0_ contract.
The length of the output contract indicates that it is a P2SH subtype.
The last item in the spending arguments is popped off, hashed with SHA256, compared against the 32-byte hash in output contract, and deserialized:

    0 <pubKey1> <pubKey2> 2 CHECKMULTISIG

The script is executed with the remaining data from the spending arguments:

    0 <signature1> 1 <pubKey1> <pubKey2> 2 CHECKMULTISIG

### Witnet address format

A Witnet address is a [Bech32](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki#bech32) encoding of:

* The human-readable part, `wit` for mainnet and `twit` for testnet.
* The data-part values:
** 1 byte: the output contract type.
** A conversion of the output contract bytes to base32.

#### Decoding

Software interpreting a Witnet address:

* MUST verify that the human-readable part is `wit` for mainnet and `twit` for testnet.
* MUST verify that the first decoded data value (the output contract type) is between `0` and `127`.
* Convert the rest of the data to bytes.

Decoders SHOULD enforce known-length restrictions on witness programs. For example, if the output contract type is `0` but the contract is neither 20 nor 32 bytes, the script must fail.

As a result of the previous rules, addresses for mainnet output contracts of type `0` are always 43 or 63 characters, but software implementing decoding of Witnet addresses MUST support any output contract type.

## Acknowledgements

* Eric Lombrozo, Johnson Lau and Pieter Wuille for [BIP-0141](https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki).
* Pieter Wuille and Greg Maxwell for [BIP-0173](https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki).
* Rusty Russell for his [Bitcoin Generic Address Format Proposal](https://rusty.ozlabs.org/?p=578).
* Mark Friedenbach for his [base32](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2014-February/004402.html) proposal.