<pre>
  WIP: WIP-adansdpc-hdwallets
  Layer: Applications
  Title: Hierarchical Deterministic Wallets
  Author: Adán SDPC <adan@witnet.foundation>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/witnet/wips/wiki/Comments:WIP-0001
  Status: Draft
  Type: Standards Track
  Created: 2018-05-07
  License: BSD-2-Clause
</pre>

## Abstract

This document describes hierarchical deterministic wallets (or "HD Wallets"): wallets which can be shared partially or entirely with different systems, each with or without the ability to spend coins.

The specification is intended to set a standard for deterministic wallets that can be interchanged between different clients. Although the wallets described here have many features, not all are required by supporting clients.

The specification consists of two parts. In a first part, a system for deriving a tree of keypairs from a single seed is presented. The second part demonstrates how to build a wallet structure on top of such a tree. 

## Copyright

This WIP is heavily based on Bitcoin's [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP-0043](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) and [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki). Therefore it inherits their original licensing: the 2-clause BSD license.

## Motivation

Deterministic wallets used in other blockchain projects like Bitcoin have been proven to be very effective by leveraging elliptic curve mathematics to permit schemes where one can calculate the public keys without revealing the private keys. This permits for example a webshop business to let its webserver generate fresh addresses (public key hashes) for each order or for each customer, without giving the webserver access to the corresponding private keys (which are required for spending the received funds).

Hierarchical deterministic wallets, which this document specifically deals with, build on that concept and improve it by allowing such selective sharing by supporting multiple keypair chains, derived from a single root. 

This document tries to translate and adapt the key derivation mechanisms described by Pieter Wuille et. al. in [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki), [BIP-0043](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) and [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) to the Witnet blockchain. 

## Specification: Key derivation

### Conventions

In the rest of this text we will assume the public key cryptography used in Witnet, namely elliptic curve cryptography using the field and curve parameters defined by [secp256k1](http://www.secg.org/sec2-v2.pdf). Variables below are either:
* Integers modulo the order of the curve (referred to as `n`).
* Coordinates of points on the curve.
* Byte sequences.

Addition (`+`) of two coordinate pair is defined as application of the EC group operation.
Concatenation (`||`) is the operation of appending one byte sequence onto another.

As standard conversion functions, we assume:
* `point(p)`: returns the coordinate pair resulting from EC point multiplication (repeated application of the EC group operation) of the `secp256k1` base point with the integer `p`.
* `ser32(i)`: serialize a 32-bit unsigned integer i as a 4-byte sequence, most significant byte first.
* `ser256(p)`: serializes the integer p as a 32-byte sequence, most significant byte first.
* `serP(P)`: serializes the coordinate pair `P = (x,y)` as a byte sequence using `SEC1`'s compressed form: (`0x02` or `0x03`) `|| ser256(x)`, where the header byte depends on the parity of the omitted y coordinate.
* `parse256(p)`: interprets a 32-byte sequence as a 256-bit number, most significant byte first.

### Extended keys

In what follows, we will define a function that derives a number of child keys from a parent key. 
In order to prevent these from depending solely on the key itself, we extend both private and public keys first with an extra 256 bits of entropy. 
This extension, called the chain code, is identical for corresponding private and public keys, and consists of 32 bytes.

We represent an extended private key as `(k, c)`, with `k` the normal private key, and `c` the chain code. 
An extended public key is represented as `(K, c)`, with `K = point(k)` and `c` the chain code.

Each extended key has 2<sup>31</sup> normal child keys, and 2<sup>31</sup> hardened child keys. 
Each of these child keys has an index. The normal child keys use indices `0 through 2<sup>31</sup>-1. 
The hardened child keys use indices 2<sup>31</sup> through 2<sup>32</sup>-1. To ease notation for hardened key indices, a number i<sub>H</sub> represents i+2<sup>31</sup>.

Hardened extended private keys are much less useful than normal extended private keys. 
However, they offer interesting security properties. They create a firewall through which multi-level key derivation compromises are impossible. Because hardened child extended public keys cannot generate grandchild chain codes on their own, the compromise of a parent extended public key cannot be combined with the compromise of a grandchild private key to create great-grandchild extended private keys. 

### Child key derivation (CKD) functions

Given a parent extended key and an index `i`, it is possible to compute the corresponding child extended key. 
The algorithm to do so depends on whether the child is a hardened key or not (or, equivalently, whether `i` ≥ 2<sup>31</sup>), and whether we're talking about private or public keys.

### Private parent key → private child key

The function CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i) &rarr; (k<sub>i</sub>, c<sub>i</sub>) computes a child extended private key from the parent extended private key:
* Check whether i ≥ 2<sup>31</sup> (whether the child is a hardened key).
** If so (hardened child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = 0x00 || ser<sub>256</sub>(k<sub>par</sub>) || ser<sub>32</sub>(i)). (Note: The 0x00 pads the private key to make it 33 bytes long.)
** If not (normal child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(point(k<sub>par</sub>)) || ser<sub>32</sub>(i)).
* Split I into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* The returned child key k<sub>i</sub> is parse<sub>256</sub>(I<sub>L</sub>) + k<sub>par</sub> (mod n).
* The returned chain code c<sub>i</sub> is I<sub>R</sub>.
* In case parse<sub>256</sub>(I<sub>L</sub>) ≥ n or k<sub>i</sub> = 0, the resulting key is invalid, and one should proceed with the next value for i. (Note: this has probability lower than 1 in 2<sup>127</sup>.)

The HMAC-SHA512 function is specified in [RFC 4231](http://tools.ietf.org/html/rfc4231).

### Public parent key → public child key

The function CKDpub((K<sub>par</sub>, c<sub>par</sub>), i) &rarr; (K<sub>i</sub>, c<sub>i</sub>) computes a child extended public key from the parent extended public key. It is only defined for non-hardened child keys.
* Check whether i ≥ 2<sup>31</sup> (whether the child is a hardened key).
** If so (hardened child): return failure
** If not (normal child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(K<sub>par</sub>) || ser<sub>32</sub>(i)).
* Split I into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* The returned child key K<sub>i</sub> is point(parse<sub>256</sub>(I<sub>L</sub>)) + K<sub>par</sub>.
* The returned chain code c<sub>i</sub> is I<sub>R</sub>.
* In case parse<sub>256</sub>(I<sub>L</sub>) ≥ n or K<sub>i</sub> is the point at infinity, the resulting key is invalid, and one should proceed with the next value for i.

### Private parent key → public child key

The function N((k, c)) &rarr; (K, c) computes the extended public key corresponding to an extended private key (the "neutered" version, as it removes the ability to sign transactions).
* The returned key K is point(k).
* The returned chain code c is just the passed chain code.

To compute the public child key of a parent private key:
* N(CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i)) (works always).
* CKDpub(N(k<sub>par</sub>, c<sub>par</sub>), i) (works only for non-hardened child keys).
The fact that they are equivalent is what makes non-hardened keys useful (one can derive child public keys of a given parent key without knowing any private key), and also what distinguishes them from hardened keys. The reason for not always using non-hardened keys (which are more useful) is security; see further for more information.

### Public parent key → private child key

This is not possible.

### The key tree

The next step is cascading several CKD constructions to build a tree. We start with one root, the master extended key m. By evaluating CKDpriv(m,i) for several values of `i`, we get a number of level-1 derived nodes. As each of these is again an extended key, CKDpriv can be applied to those as well.

To shorten notation, we will write CKDpriv(CKDpriv(CKDpriv(m,3<sub>H</sub>),2),5) as m/3<sub>H</sub>/2/5. Equivalently for public keys, we write CKDpub(CKDpub(CKDpub(M,3),2),5) as M/3/2/5. This results in the following identities:
* N(m/a/b/c) = N(m/a/b)/c = N(m/a)/b/c = N(m)/a/b/c = M/a/b/c.
* N(m/a<sub>H</sub>/b/c) = N(m/a<sub>H</sub>/b)/c = N(m/a<sub>H</sub>)/b/c.
However, N(m/a<sub>H</sub>) cannot be rewritten as N(m)/a<sub>H</sub>, as the latter is not possible.

Each leaf node in the tree corresponds to an actual key, while the internal nodes correspond to the collections of keys that descend from them. The chain codes of the leaf nodes are ignored, and only their embedded private or public key is relevant. Because of this construction, knowing an extended private key allows reconstruction of all descendant private keys and public keys, and knowing an extended public keys allows reconstruction of all descendant non-hardened public keys.

### Key identifiers

Extended keys can be identified by the first 20 bytes in the result of applying the SHA256 hash function on the serialized ECDSA public key `K`, ignoring the chain code.

The first 4 bytes of the identifier are called the key fingerprint.

### Path levels

We define the following 5 levels in key tree paths:

```
m / purpose' / coin_type' / account' / change / address_index
```

Apostrophe in the path indicates that hardened derivation is used.

Each level has a special meaning, described below.

#### Purpose

Purpose is a constant set to `3'` (or `0x80000003`). It indicates that the subtree of this node is used according to this specification.

Hardened derivation is used at this level.

#### Coin type

One master node (seed) can be used for unlimited number of independent cryptocoins such as Bitcoin, Litecoin or Namecoin. 
However, sharing the same space for various cryptocoins has some disadvantages.

This level creates a separate subtree for every cryptocoin, avoiding reusing addresses across cryptocoins and improving privacy issues.

Coin type is a constant set to `4919'` (or `0x80001337`) for Witnet. The list of already allocated coin types can be found in [SLIP-0044](https://github.com/satoshilabs/slips/blob/master/slip-0044.md#registered-coin-types).

Hardened derivation is used at this level. 

#### Account

This level splits the key space into independent user identities, so the wallet never mixes the coins across different accounts.

Users can use these accounts to organize the funds in the same fashion as bank accounts; for donation purposes (where all addresses are considered public), for saving purposes, for common expenses etc.

Accounts are numbered from index 0 in sequentially increasing manner. This number is used as child index in BIP32 derivation.

Hardened derivation is used at this level.

Software should prevent a creation of an account if a previous account does not have a transaction history (meaning none of its addresses have been used before).

Software needs to discover all used accounts after importing the seed from an external source. Such an algorithm is described in [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki#account-discovery).

#### Change

Constant 0 is used for external chain and constant 1 for internal chain (also known as change addresses). External chain is used for addresses that are meant to be visible outside of the wallet (e.g. for receiving payments). Internal chain is used for addresses which are not meant to be visible outside of the wallet and is used for return transaction change.

Public derivation is used at this level. 

#### Index

Addresses are numbered from index 0 in sequentially increasing manner. This number is used as child index in BIP32 derivation.

Public derivation is used at this level. 

### Serialization format
    
Extended public and private keys are serialized as follows:

| Size     | Description |
|----------|-------------|
| 1 byte   | Depth: `0x00` for master nodes, `0x01` for level-1 derived keys, etc. |
| 4*depth bytes | Serialized path; each entry is encoded as 32-bit unsigned integer, big endian |
| 32 bytes | The chain code |
| 33 bytes | The public key or private key data |

The key data field will be ser<sub>P</sub>(K) for public keys and `0x00` || ser<sub>256</sub>(k) for private keys. 

This structure can be encoded using Bech32 format described in [WIP-adansdpc-addresses](). We will use 'xpub' human-readable part for extended public keys and 'xprv' for extended private keys.

When importing a serialized extended public key, implementations must verify whether the `X` coordinate in the public key data corresponds to a point on the curve. If not, the extended public key is invalid.

### Master key generation

The total number of possible extended keypairs is almost 2<sup>512</sup>, but the produced keys are only 256 bits long, and offer about half of that in terms of security. Therefore, master keys are not generated directly, but instead from a potentially short seed value.

* Generate a seed byte sequence `S` of a chosen length (between 128 and 512 bits; 256 bits is advised) from a (P)RNG.
* Calculate `I = HMAC-SHA512(Key = "Witnet seed", Data = S)`
* Split `I` into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* Use parse<sub>256</sub>(I<sub>L</sub>) as master secret key, and I<sub>R</sub> as master chain code.
In case I<sub>L</sub> is 0 or ≥n, the master key is invalid.

## Specification: Wallet structure

The previous sections specified key trees and their nodes. The next step is imposing a wallet structure on this tree. 
The layout defined in this section is a default only, though clients are encouraged to mimic it for compatibility, even if not all features are supported.

### The default wallet layout

An HDW is organized as several 'accounts'. Accounts are numbered, the default account being number `0`. 
Clients are not required to support more than one account - if not, they only use the default account.

Each account is composed of two keypair chains: an internal and an external one. The external keychain is used to generate new public addresses, while the internal keychain is used for all other operations (change addresses, generation addresses, ..., anything that doesn't need to be communicated). 
Clients that do not support separate keychains for these should use the external one for everything.
* m/3'/4919'/i'/0/k corresponds to the `k`th keypair of the external chain of account number `i` of the HDW derived from master `m`.
* m/3'/4919'/i'/1/k corresponds to the `k`th keypair of the internal chain of account number `i` of the HDW derived from master `m`.

### Use cases

Use cases for HDW are described in detail in [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

## Compatibility
   
To comply with this standard, a client must at least be able to import an extended public or private key, to give access to its direct descendants as wallet keys. 
The wallet structure (master/account/chain/subchain) presented in the second part of the specification is advisory only, but is suggested as a minimal structure for easy compatibility - even when no separate accounts or distinction between internal and external chains is made. 
However, implementations may deviate from it for specific needs; more complex applications may call for a more complex tree structure.

## Security

In addition to the expectations from the EC public-key cryptography itself:
* Given a public key `K`, an attacker cannot find the corresponding private key more efficiently than by solving the EC discrete logarithm problem (assumed to require 2<sup>128</sup> group operations).

The intended security properties of this standard are:
* Given a child extended private key (k<sub>i</sub>,c<sub>i</sub>) and the integer i, an attacker cannot find the parent private key k<sub>par</sub> more efficiently than a 2<sup>256</sup> brute force of HMAC-SHA512.
* Given any number (2 ≤ N ≤ 2<sup>32</sup>-1) of (index, extended private key) tuples (i<sub>j</sub>,(k<sub>i<sub>j</sub></sub>,c<sub>i<sub>j</sub></sub>)), with distinct i<sub>j</sub>'s, determining whether they are derived from a common parent extended private key (i.e., whether there exists a (k<sub>par</sub>,c<sub>par</sub>) such that for each j in (0..N-1) CKDpriv((k<sub>par</sub>,c<sub>par</sub>),i<sub>j</sub>)=(k<sub>i<sub>j</sub></sub>,c<sub>i<sub>j</sub></sub>)), cannot be done more efficiently than a 2<sup>256</sup> brute force of HMAC-SHA512.
Note however that the following properties does not exist:
* Given a parent extended public key (K<sub>par</sub>,c<sub>par</sub>) and a child public key (K<sub>i</sub>), it is hard to find i.
* Given a parent extended public key (K<sub>par</sub>,c<sub>par</sub>) and a non-hardened child private key (k<sub>i</sub>), it is hard to find k<sub>par</sub>.

### Implications

Private and public keys must be kept safe as usual. Leaking a private key means access to coins and contracts—leaking a public key can mean loss of privacy.

Somewhat more care must be taken regarding extended keys, as these correspond to an entire (sub)tree of keys.

One weakness that may not be immediately obvious, is that knowledge of a parent extended public key plus any non-hardened private key descending from it is equivalent to knowing the parent extended private key (and thus every private and public key descending from it). This means that extended public keys must be treated more carefully than regular public keys. It is also the reason for the existence of hardened keys, and why they are used for the account level in the tree. This way, a leak of account-specific (or below) private key never risks compromising the master or other accounts.

## Acknowledgements

* Gregory Maxwell for the original idea of type-2 deterministic wallets, and many discussions about it.
* Alan Reiner for the implementation of this scheme in Armory, and the suggestions that followed from that.
* Pieter Wuille for authoring and publishing [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).
* Eric Lombrozo for reviewing and revising [BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).
* Marek Palatinus and Pavol Rusnak for authoring and publishing [BIP-0043](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) and [BIP-0044](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki).
* Pavol Rusnak for authoring and publishing [SLIP-0032](https://github.com/satoshilabs/slips/blob/master/slip-0032.md).