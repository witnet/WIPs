<pre>
    WIP: WIP-mariocao-adjust-mining-probability
    Layer: Consensus (hard fork)
    Title: Adjust mining probability
    Authors: Mario Cao <mario@witnet.foundation>
             Gorka Irazoqui <gorka.irazoki@gmail.com>
    Discussions-To: `#dev-general` channel on Witnet Community's Discord server
    Status: Draft
    Type: Standards Track
    Created: 2021-01-26
    License: BSD-2-Clause
</pre>


## Abstract

This proposal adjusts the mining algorithm to limit the probability of non-reputed and non-active nodes mining a block.

The goal of this proposal is to mitigate potential network abuses that could be carried by malicious users controlling a large number of nodes. This could pose a serious threat to the network availability and security.


## Motivation and Rationale

after more than 3 months of the Witnet mainnet launch, the network has faced episodes of a bigger transaction latency than expected. In periods with a mempool full of transactions, blocks not using the complete space appeared in the Witnet network. This reduces the efficiency of the system, thus increasing indirectly the transaction latency perceived by users. Under a more critical scenario, this problem could pose an availability threat in terms of a [denial-of-service attack][DoS], in which a large number of malicious nodes could be mining empty blocks or, even, endanger the network stability by affecting internal network metrics (e.g. Active Reputation Set).

A thorough investigation concluded that the majority of these blocks (if not all) were produced by non-reputed and non-active nodes (i.e. newcomers to the network). Regardless of the intentionality of this behaviour, these blocks were appearing in a much higher volume than the newcomer mining probability that was designed in the protocol.

The current mining algorithm can be summarized as follows:

1. Miners build a candidate block.
2. Miners compute their eligibility by using the [VRF]-based [RandPoE] algorithm with by using the ARS census as the difficulty parameter (labelled as N).
3. If eligible (and no better candidate was received), miners will broadcast their candidate blocks.

Upon reception of multiple candidates, the winner will be selected as follows:
1. Candidate blocks from reputed nodes are prioritized over those from non-reputed nodes.
2. If there are still multiple candidates, the one with the lowest VRF is chosen.

The main problem with the aforementioned algorithm is that the eligibility is computed using the ARS as census, but the prioritization is only taking into account the reputed nodes. This led to the increase of the newcomer mining probability. For example, the 18th of January the newcomer mining probability could be computed with the following network conditions:

```
TRS(Total Reputation Set)  = 2000 nodes
ARS(Active Reputation Set) = 6000 nodes
RF(Replication Factor) = 3

P(non-reputed node mines a block) = P(none reputed node mines a block) = P(single reputed does not mine a block)^TRS = (1- P(single reputed mine a block))^TRS = (1 - RF/ARS)^TRS ~= 37%
```

In the initial mining algorithm design [rand-poe], DoS attacks were carefully taken into account. An upper bound for the probability of an attacker censoring blocks was set to approximately 2%. However, before the mainnet launch some mining parameters were changed to achieve a more fair block reward distribution. It was assummed that the reputation expiration would be slower compared to the activity period (related to the ARS). This assumption did not hold in mainnet as seen in the former example. This mistake led to a significant increase in the newcomer mining probability.

Summing up, there is a need for adjusting the newcomer mining probability to mitigate the impact of potential DoS attacks that may endanger the availability and the security of the Witnet network.


## Proposal / Specification

This document proposes to use the same census for the eligibility and the prioritization as initially designed. The previous candidate selection will be modified as follows:

Upon reception of multiple candidates:
1. Candidate blocks from reputed nodes are prioritized over those from non-reputed nodes.
2. Candidate blocks from active nodes are prioritized over those from non-active nodes.
3. If there are still multiple candidates, the one with the lowest VRF is chosen.

With the previous changes, the newcomer mining probability is bounded as initially designed in [RandPoe] while the fair distribution over those nodes that have proven some contribution to the network service and security still applies.


## Reference Implementation

The business logic described in the specification has been implemented in [witnet-rust] by [pull request #1829][witnet/witnet-rust#1829].


## Acknowledgments

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community [witnet/witnet-rust#1827].

Special thanks to [Luis Rubio][lrubiorod] for the reference implementation in Rust.


[DoS]: https://en.wikipedia.org/wiki/Denial-of-service_attack
[lrubiorod]: https://github.com/lrubiorod
[rand-poe]: https://github.com/witnet/research/blob/master/reputation/docs/randpoe.md
[VRF]:  https://medium.com/witnet/c847edf123f7
[witnet-rust]: https://github.com/witnet-rust
[witnet/witnet-rust#1827]: https://github.com/witnet/witnet-rust/issues/1827
[witnet/witnet-rust#1829]: https://github.com/witnet/witnet-rust/issues/1829
