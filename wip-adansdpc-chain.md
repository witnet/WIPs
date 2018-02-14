```
  WIP: WIP-adansdpc-chain
  Title: Witnet block chain
  Author: Adán SDPC <adan@witnet.foundation>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/witnet/wips/wiki/Comments:WIP-adansdpc-chain
  Status: Draft
  Created: 2018-02-12
  License: BSD-2-Clause
```


## Abstract

The purpose of this document is offering the community a first proposal and implementation for an important part of the technical specifics that the original [Witnet whitepaper](https://arxiv.org/pdf/1711.09756.pdf) left to be defined.

This WIP focuses on the characteristics of the Witnet block chain. Subsequent WIPs will address other fundamental parts like transaction types, serialization formats, key chains, wallets and addresses.

Following the publication of this document as a draft, discussion on its contents should be conducted on a public and open way.
In accordance to [WIP-0001](wip-0001.md), the author(s) of this WIP is responsible for shepherding the discussions in the appropiate forums and attempt to build community consensus about the proposal.

## Copyright

This WIP is published under the conditions of the BSD 2-clause license.

## Motivation

The first version of the [Witnet whitepaper](https://arxiv.org/pdf/1711.09756.pdf), originally submitted to arXiv on 27 Nov 2017, lays the foundation of Witnet: a decentralized oracle network protocol.

The whitepaper is rather extensive on the functioning of the Witnet block chain, the underlying network and its ruling protocol itself.
However, the authors decided to omit technical specifications for a first implementation of the protocol.
The reasons behind these decisions were:

- Conciseness: the authors preferred to focus on problem analysis, economics and game theory rather than setting forth any technical specifics.
- Governance: Witnet is aimed to be an open source development in which the community plays a key role.
Defining those specifics without an open, public discussion would contravene that openness principle.

This WIP addresses the characteristics of the Witnet block chain, proposes a specification and explains the rationale behind the design.

## Rationale

### Introduction

There are two main reasons why a distributed ledger needs a block chain data structure.
And these reasons are hidden in plain sight right in the name of the structure itself: "block" and "chain".

- **Block**: the essential purpose for blocks is bundling transactions into chunks that can be more easily and atomically agreed upon when trying to build consensus on the current state of the ledger (_"get everyone on the same page"_).

- **Chain**: chaining the blocks creates an ordered record that evinces the order in which they were generated, gives them a rough timestamp and makes them difficult to modify afterwards by imposing some requirements on blocks being added to the chain.

Those requirements imposed to new blocks can be divided into **formal requirements** and **power requirements**.

#### Formal requirements

All transactions and blocks in a block chain must comply with some formats.
If they succeed to do so, they are considered "valid".

An example of formal requirement is the block size limit found in most block chains.

#### Power requirements

Every competent participant in a block chain network shall be capable of building formally valid blocks.
However, if a number of them produce conflicting blocks at the same time, a consensus problem arises
There needs to be some way to decide who has an exclusive right to produce new blocks, or at least to know who has the last word on how to resolve the conflict.

This is were power requirements come into play.

The most popular form of this kind of requirements are Proof-of-Work (PoW) algorithms.
Participants in a PoW block chain agree to only accept the blocks (or chain of blocks) containing a proof showing that the author of the block (a.k.a _the miner_) was the participant who committed the most effort to solve a cryptographic challenge.
This _effort_ is not other than devoting computational resources—namely, spending electrical power.

The point is that, as miners normally earn rewards for producing valid blocks, they find more economic incentives in allocating their resources on creating new blocks than on rewriting the existing chain.
This way, as a block gets "burried under" more blocks in the chain, the economical cost of modifying them gets higher and higher, to the point where such a rewrite becomes prohibitive and therefore the blocks can be considered to be immutable.

Generally speaking, PoW aligns "chain power" with computational power.

### Power requirements in Witnet

The Witnet block chain is aimed to provide a mechanism that allows any piece of software to request the **retrieval**, **attestation** and **delivery** (R-A-D) of publicly available information without relying in any centralized authority.

This kind of construction, which the Witnet whitepaper names as "Decentralized Oracle Network" (DON), requires the attesting parties (called _witnesses_) to be completely independent from each other.
Otherwise, they could easily collude and tamper with a R-A-D request with the view to manipulate the outcome of a smart contract in which they had some partisan interest.

This fact makes PoW schemes unsuitable and undesirable for DONs. Power cannot be aligned with computational power.

Instead, power must be aligned with the "reputation" of the witnesses. This is, their past performance in terms of honesty.
The longer they remain honest and perform RAD requests without cheating, the better reputation score they will have and the more likely they will earn the right to (1) resolve future R-A-D tasks, and (2) append a new block into the chain, earning fees in both cases.

Reputation points are a zero-sum game. The sum of the reputation scores from all the participants in the network is always the same.
In the ideal scenario that none of the witnesses ever cheated, they will all remain with a neutral reputation value.
Only when witnesses misbehave a certain part of their reputation is deducted from their score and distributed among their honest peers.

This scheme, which can be regarded as a form of Proof-of-Stake, creates long-term incentives for miners (called witnesses here) to be honest and not to collude with others to deceive the rest of the network.

As discussed in sections 3 and 4 of the whitepaper, there exists a "double-agent incentive" that guarantees that in the event that someone was trying to coordinate witnesses outside of the network, witnesses would prefer to lie to the coordinator and still fulfill their duties honestly.

### Implications of PoS for the topology of the chain

The reputation-based Proof-of-Stake scheme proposed in the Witnet whitepaper has important implications for the topology of the Witnet block chain.

#### Block time

In the Witnet chain, the time between creation of new blocks does not depend on a probabilistic process (the time spent by the fastest miner on solving the PoW challenge). Instead, it is a deterministic routine—blocks are created periodically, regardless of them being totally full or completely empty.

Checkpoints are created at regular intervals, and the time between checkpoints is referred to as **epoch**.

Epochs are fixed in time. They start and finish at the same time for all the participants in the network.

At any given epoch _t_, all users have access to L~t, a snapshot of the ledger at the preceding checkpoint _t_. Epoch _t_ begins at checkpoint _t_ and finishes at checkpoint _t + 1_. In other words, checkpoint _t_ closes epoch _t − 1_ and opens epoch _t_.

In PoW chains, there exists a lower bound to block time for a number of reasons:

- We need to ensure that at least a certain amount of work is committed to every block. Otherwise, the immutability principle would be broken.
- If blocks come too often, they lose their point and we would better just chain transactions and forget about blocks altogether.
- There is a minimum time that a new block takes to propagate over the network. As every block needs to be built upon previous one, extremely short block times cause miners to waste a lot of resources trying to mine upon blocks that may have already been buried by blocks created by other miners.

PoW chains use some sensible block time value that takes all those factors into account and use some "difficulty retarget" mechanism that periodically adjusts the power requirements so the block time remains stable despite of oscillations in the computational power of the miners.

In Witnet, a lower bound to epoch time length exists too for similar reasons, but instead of being related to oscillations in power, it is rather related to expectable differences between the clocks of the participants of the network.

As epochs are fixed in time, two participant nodes with internal clocks many seconds apart from each other may believe they are living on different epochs.
Thus, the epoch length must be always be greater than the natural time drift that may reasonably exist between the clocks of all the participant nodes in a broadly adopted peer-to-peer network.

#### Avoiding double-spend

The Witnet protocol's approach to avoiding double-spend is similar to Bitcoin's insofar as the block chain is used as a timestamping calendar that marks the order of the transactions: coins spent during past epochs cannot be spent again.

The problem comes when someone produces two different valid transactions trying to spend the same coin. Here is where Witnet differs radically from Bitcoin and other block chains.

The Bitcoin protocol does not establish what miners should do if they receive two conflicting transactions.
Given the existing incentives, it is to be expected that they include the most profitable transaction (the one paying more fees per weight) into a block and drop the other.
But they can also at their sole discretion just drop both transactions and focus on other non-controversial transactions.
The point is that every miner can implement their own policy on how to act in those cases.

On the contrary, the Witnet protocol is explicit on how to address double-spend attempts: they are welcome.

At each epoch, each miner has a certain chance of eligibility for creating a new block (_epoch leadership_).
That chance is proportional to their "influence": this is, their share of the total reputation score in the network.

Epoch leadership can be seen as a sort of raffle in which the more honest you are, the more chances of winning you have.
But this raffle has a certain particularity: the tickets don't have only a sole number printed on them.
Instead, they have a range of numbers, and the more honest you are, the broader your range of numbers is.

The specific algorithm for choosing miners for each epoch is explained in section 5 (figures 12 and 13) of the [Witnet whitepaper](https://arxiv.org/pdf/1711.09756.pdf) and will be put forth and discussed in a future WIP document.

The point here is that the algorithm "elects" one miner for each epoch, but, here comes the trick: only **on average**.

- For some epochs, there will be a chance that no participant "had a ticket with the raffle winning number".
- For other epochs, the winning number will fall inside the range of numbers in the ticket of more than one participant.

The first case is dealt easily by running the miner election algorithm as many times as needed in search for "fallback miners".
More detail on this strategy can be found in section 5.1 (figure 14) of the [Witnet whitepaper](https://arxiv.org/pdf/1711.09756.pdf) and will be put forth and discussed in a future WIP document.

The second case is the one particularly affecting the topology of the Witnet block chain.

If two or more miners are elected for the same epoch, it is to be expected that two or more valid blocks will coexist for the same epoch.

This is solved in the Witnet protocol in a novel way:

- The Witnet block chain is not a linear chain as such.
- The values in transaction outputs are not absolute but relative.

Instead, from a topological perspective, it is a more generalized type of [_Directed Acyclic Graph_](https://en.wikipedia.org/wiki/Directed_acyclic_graph).

Blocks are identified not only for their hash (which is derived from its contents) but also for their epoch/checkpoint. Two or more blocks for the same epoch can coexist as long as (1) they are correctly formed, and (2) the miners provide valid proofs of their epoch leadership inside the blocks' headers.

Given two or more blocks for the same checkpoint _t_, they will likely contain a similar (if not the same) set of transactions that were broadcast to the network during epoch _t − 1_.

Such transaction duplicity is addressed in the most straightforward way possible: a number _n_ of transactions spending the same output with value _v_ in the same block or in a different block for the same checkpoint are perfectly valid, but the value is evenly split among their coincident inputs.
This is, each of the _n_ transactions will be able to spend at most _v / n_ coins.

To made this possible, Witnet transactions have a series of nice properties explained in section 6 of the [Witnet whitepaper](https://arxiv.org/pdf/1711.09756.pdf). Those properties will also be put forth and discussed in a future WIP document.

## Specification

### Constants

- **Epoch duration**: 90 seconds.

### Blocks

Note: `flex-n` types are defined in the [Flex encoding](#flex-encoding) section below.

#### Block structure

| Field | Description | Type |
|---|---|---|
| Block size | Number of bytes following up to end of block | flex-2 |
| Block header | The header of the block | structure |
| Transaction counter | Number of transactions in the block | flex-2 |
| Transactions | A non-empty list of transactions | \[structure\] |

#### Block header structure

| Field | Description | Size |
|---|---|---|
| Version | Block version number | flex-1 |
| Beacon | A checkpoint beacon for the epoch that this block is closing | structure |
| hashMerkleRoot | 256-bit hash based on all of the transactions committed to this block | u256 integer |
| Proof | A miner-provided proof of leadership | structure |

#### Proof of leadership structure

| Field | Description | Size |
|---|---|---|
| BlockSig | An enveloped signature of the block header except the Proof part | structure |
| PowerSig | An enveloped signature of the epoch beacon | structure |
| Influence | The miner influence as of last checkpoint | flex-1-shift-2 |

#### Checkpoint beacon structure

| Field | Description | Size |
|---|---|---|
| Checkpoint | The serial number for an epoch | flex-1-shift-2 |
| hashPrevBlock | 256-bit hash of the previous block header | u256 integer |

### Signatures

#### Enveloped signature structure

| Field | Description | Size |
|---|---|---|
| CryptoSys | A cryptosystem identifier | flex-b1 |
| Signature | A digital signature | structure |

#### Cryptosystem identifiers registry

| Dec | Hex | Flex-b1 | Cryptosystem | Signature size |
|---|---|---|---|---|
| 0 | 0x00 | 0000 | None | 0 bytes |
| 1 | 0x01 | 0001 | ECDSA over secp256k1 | 48 bytes |

Future WIPs can propose addition of new entries to this registry.

### Flex encoding

Flex encoding saves disk space and bandwidth by optimizing the size of variable types.

It sacrifices part of the range of typical multi-byte unsigned integer types (u16, u32, u64) by using their first bits for the special purpose of signaling how many bytes to read afterwards.

This type of encoding is specially convenient for all kind of variables which values typically fall in the beginning of their possible range.
That is the case for counters, magic numbers, versions, etc.

Even for random numbers uniformly generated from the effective domain of each type, this encoding provides a small saving (likely under 3%).
When used extensively, this small space saving builds up and on the long term it can save MBs or GBs to the total block chain size, which is specially benefitial to speed up initial synchronization of full nodes.   

#### Flex types

All flex types are named like `flex-n`, where `n` is the number of bits used for encoding the byte length.

For each of the flex types, the total size in bytes ranges from `1` to `2^n`, and the maximum value is `2^(2^(n + 3) - n) - 1`.

##### flex-1

| First bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|
| 0 | 7 bits  | 127    | 1 byte  | 50% |
| 1 | 15 bits | 32_767 | 2 bytes | 0%  |

Uniform random saving: 0.193%

##### flex-2

| First bit | Second bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|---|
| 0 | 0 | 6 bits  | 63            | 1 byte  | 75% |
| 0 | 1 | 14 bits | 16_383        | 2 bytes | 50% |
| 1 | 0 | 22 bits | 4_194_303     | 3 bytes | 25% |
| 1 | 1 | 30 bits | 1_073_741_823 | 4 bytes | 0%  |

Uniform random saving: 0.098%

##### flex-3

| First bit | Second bit | Third bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|---|---|
| 0 | 0 | 0 | 5 bits  | 31                        | 1 byte  | 89.5% |
| 0 | 0 | 1 | 13 bits | 8_191                     | 2 bytes | 75%%  |
| 0 | 1 | 0 | 21 bits | 2_097_151                 | 3 bytes | 62.5% |
| 0 | 1 | 1 | 29 bits | 536_870_912               | 4 bytes | 50%   |
| 1 | 0 | 0 | 37 bits | 137_438_953_472           | 5 bytes | 37.5% |
| 1 | 0 | 1 | 45 bits | 35_184_372_088_831        | 6 bytes | 25%   |
| 1 | 1 | 0 | 53 bits | 9_007_199_254_740_991     | 7 bytes | 12.5% |
| 1 | 1 | 1 | 61 bits | 2_305_843_009_213_693_951 | 8 bytes | 0%    |

Uniform random saving: 0.049%

#### Flex-shift encoded types

Flex-shift encoding offers even more flexibility over normal flex encoding.

Take for example the checkpoint number in block headers.
Using `flex-2` encoding, we would be able to encode its value using only 1 byte for the first 63 epochs.
Then, it would take 2 bytes for the next 16K epochs; 3 bytes for the next 4M; and finally 4 bytes until the variable overflowed over 1 billion epochs later.
Assuming a pre-established epoch duration of 90 seconds, we would be taking advantage of the following space savings:

- 75% during 1 hour and a half.
- 50% during 17 days.
- 25% during 16 years.

A 25% space saving (1 byte out of 4) for 16 years is a good deal, but the bigger savings (75%, 50%) last a very short period and their impact is thus very limited.

Flex-shift types give up the early space saving percentages in favor of the later space savings being applicable to a wider range from their domain.

In the example above, using `flex-1-shift-2` would offer a constant 25% space saving (1 byte out of 4) during the first 24 years.
Only then would the variable start to take all the 4 bytes.

All flex-shift types are named like `flex-n-shift-m`, where `n` is the number of bits used for encoding the byte length, and `m` is the shift number.

For each of the flex-shift types, the total size in bytes ranges from `m + 1` to `(2^n) + m`, and the maximum value is `2^(8 × (2^n + m) - n) - 1`.

Note that normal flex types are the same as their flex-shift equivalents with shift number `m = 0`.
This is, `flex-3-shift-0` is the same as `flex-3`.

##### flex-1-shift-1

| First bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|
| 0 | 15 bits | 32_767    | 2 byte  | 33% |
| 1 | 23 bits | 8_388_607 | 3 bytes | 0%  |

Uniform random saving: 0.129%

##### flex-1-shift-2

| First bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|
| 0 | 23 bits | 8_388_607     | 3 byte  | 25% |
| 1 | 31 bits | 2_147_483_647 | 4 bytes | 0%  |

Uniform random saving: 0.098%

#### Bitwise flex encoding

In certain cases, we can find values with a very limited possible range or domain.
Using whole bytes to encode them leave us with many variables stuffed with unnecessary padding (leading zeros) at the bit level.

Bitwise flex encoding allows to make the most of flex encoding to save space for values that we know to be bound to very low values, like some kind of identifiers and flags.

All bitwise flex types are named like `flex-bn`, where `n` is the number of bits used for encoding the byte length.

For each of the bitwise flex types, the total size in bits ranges from `4` to `4 × (2^n)`, and the maximum value is `2^(4 × 2^n - n) - 1`.

When serializing a structure or message containing an odd number of variables encoded in this way, the resulting stream of bits will not be byte-aligned (its length will not be a multiple of 8).
Depending on the platform, if byte-alignment is strictly necessary, a single message-level padding consisting of 4 zero bits can be affixed.

##### flex-b1
| First bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|
| 0 | 3 bits | 7   | 4 bits | 50% |
| 1 | 7 bits | 127 | 8 bits | 0%  |

Uniform random saving: 2.74%

##### flex-b2
| First bit | Second bit | Value size | Max. value | Total size | Space saving |
|---|---|---|---|---|---|
| 0 | 0 | 2 bits  | 1      | 4 bits  | 75% |
| 0 | 1 | 6 bits  | 63     | 8 bits  | 50% |
| 1 | 0 | 10 bits | 1_023  | 12 bits | 25% |
| 1 | 1 | 14 bits | 16_383 | 16 bits | 0%  |

Uniform random saving: 1.66%

## Backwards compatibility

As this WIP is the first public description of the Witnet block chain specifics, there exist no previous documents nor protocol implementations that could conflict with what is proposed here.

## Reference implementation

A reference implementation is being worked at the author's [chain-base branch](https://github.com/aesedepece/rust-witnet/tree/chain-base).