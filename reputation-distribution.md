<pre>
  WIP: WIP-00XX
  Layer: Consensus (hard fork)
  Title: Distribution of slashed collateral
  Authors: drcpu <drcpu@protonmail.com>
  Discussions-To: https://github.com/witnet/witnet-rust/discussions/2271
  Status: Draft
  Type: Standards Track
  Created: 2022-11-10
  License: BSD-2-Clause
</pre>

## Abstract

This improvement proposal aims to refine the distribution of slashed reputation to prevent abnormal changes in the reputation distribution and decrease the efficacy of reputation-stealing attacks.

## Motivation and rationale

Presently, the reputation system suffers from two major weaknesses. The first one is that nodes can sometimes earn unreasonably large amounts of reputation and the second one is that malicious node operators can manipulate the TRS through reputation-stealing attacks.

### Earning reputation

New nodes can sometimes earn a disproportionately large amount of reputation by solving a single data request. This appears to be a random process, but it does challenge the assumption of the reputation system where only nodes which have proven themselves to be thrustworthy solve the majority of data requests. The way this usually occurs is that a large set of reputation packets expires or is slashed and all this reputation is redistributed in the same epoch to a small set of nodes. Depending on the velocity of witnessing activities in the network, this can even result in a reinforcing effect because those large packets of reputation will again expire at the same moment in time. However, up until this moment, we have not really observed behavior like this.

To illustrate the distribution of reputation earned by solving a single data request, I analyzed the size of each reputation packet up to epoch 1500000. Below table is the description of a cumulative function and can be summarized as `x%` of packet has a size of `y` reputation or less.

| Packet | Size |
|--------|------|
|   10%  |    2 |
|   20%  |    3 |
|   30%  |    6 |
|   40%  |   10 |
|   50%  |   17 |
|   60%  |   31 |
|   70%  |   58 |
|   80%  |  113 |
|   90%  |  215 |
|   95%  |  329 |
|   99%  |  804 |

This table shows that, in most cases, solving a data request does not yield a disproportionate amount of reputation. For example, when a node resolves a data request, it will earn 17 reputation or less half of the time. However, 1% of the time, a node will earn 804 reputation or more. In the past 1500000 epochs, there have been roughly 27M reveals meaning there were also around 270000 cases where a reveal was rewarded with 804 reputation or more. Given the average reputation distribution, in 270000 cases, a set of nodes was propelled into the upper quarter of the TRS by solving one data request instead of finding its way there through solving multiple consecutive data requests honestly.

In order to make sure the distributed reputation packets do not become disproportionately large, whenever a large batch of reputation expires, it will need to be distributed over several epochs rather than all in one epoch.

### Reputation-stealing attacks

The second major weakness is the efficacy of reputation-stealing attacks. As the network is completely permissionless, anyone can craft data requests and have them solved by node operators. Unfortunately, this also results in malicious actors being able to craft data requests aimed at stealing reputation. Since slashed reputation is redistributed in the same epoch as in which it was slashed, malicious node operators can force other node operators to commit lies, have their reputation slashed and directly earn (part of) that slashed reputation. This type of attack can be non-discriminative and simply target any node operator or it can target specific nodes through combining deanonimization and DDOS attacks. At the moment these attacks (especially the targetted ones) are unlikely to be economically feasible and no proof has surfaced that these attacks are actively happening.

The straightforward solution to this issue is to distribute slashed reputation in the next epoch which contains data requests. By virtue of data request solving eligibility being random, no node operator can predict whether he will be able to solve any of the data requests in the epoch after he has forced other node operators to lie. Thus introducing this change significantly reduces the efficacy of such an attack.

However, distributing slashed reputation in a single epoch with data requests would again result in nodes being able to earn unreasonably large packets of reputation. Hence, we also need to redistribute the slashed reputation using a more gradual distribution spread over multiple data request epochs. While distributing slashed reputation over multiple subsequent epochs would seem to result in an increased probability of a malicious node operator profiting off its malicious data requests, the expected value of stolen reputation stays constant as demonstrated below.

First, the probability of any operator solving a data request requiring `w` witnesses is defined as `C(w, 1) * E ^ 1 * (1 - E) ^ (w - 1)` where `E` is the eligibility of the operator. If a data request requires 10 witnesses and the node operator's eligibility is `0.05%`, the probability `P` of it solving a data request is `C(10, 1) * 0.05% ^ 1 * 99.95% ^ 9 = 0.498%`.

For this example, assume a malicious operator created a data request which slashed a total sum of `2500` reputation from other operators and that each subsequent epoch will contain a single data request requiring ten witnesses. The probability of solving `y` data requests in `x` subsequent epochs is `C(x, y) * P ^ y * (1 - P) ^ (x - y)` where `P` is the probability an operator can solve a single one.

If all reputation is distributed in the next epoch containing a single data request, the previous formula simplifies to `C(1, 1) * 0.498% ^ 1 * 99.52% ^ 0 = 0.498%`. The malicious operator will thus earn an expected amount of reputation equal to `0.498% * 2500 / 10 = 1.244`.

If you spread out that reputation over `x` epochs, each epoch the operator has the same probability `P` to solve the data request. However, at the same time the reputation it can earn for solving one is equal to `2500 / x`. Take `x` equal to 5 and substitute this in the same formula: `C(5, y) * 0.498% ^ y * 99.52% ^ (5 - y)` where `y` ranges from 1 to 5. For each data request solved, the node operator can earn `2500 / 5 / 10 * y` reputation. Summing these probabilities multiplied with the reputation results in an expected reputation of `1.244` again.

## Specification

It is impossible to predict how much reputation will be slashed when one or more node operators lie or how much reputation will expire in the next epoch containing data requests. Therefore, the optimal solution to guarantee reputation packets never exceeds a certain upper limit is to accumulate the reputation in a dripping faucet. The default reputation earned for a witnessing act is one. This proposal introduces the rule that the upper limit should never exceed this default by more than five times.

Each epoch, the expired, slashed and remaining reputation from the previous epoch is added to the faucet. The amount of reputation node operators can earn that epoch is equal to the reputation in the faucet divided by the total amount of unique witnessing acts. If the result exceeds five, it is capped and the remainder is left for the next epoch. If the result is a value smaller than five, the reputation that can be earned is floored to an integer value and the remainder is, again, left for the next epoch.

This mechanism will result in a significantly more fair distribution of reputation since no node operator can earn a disproportionate amount of reputation by being eligible at the right time.

## Backwards compatibility

- Implementer: a client that implements or adopts the protocol improvements proposed in this document.
- Non-implementer: a client that fails to or refuses to implement the protocol improvements proposed in this document.

#### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions that were considered valid by former versions of the protocol MUST continue to be considered valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with those coming from implementers, that is, the new rules.
- Implementers MUST apply the proposed logic when evaluating transactions and proposed blocks.

As a result:

- Non-implementers MAY NOT get their blocks and transactions accepted by implementers.
- Implementers MAY get their valid blocks accepted by non-implementers.
- Non-implementers MAY consolidate different blocks and superblocks than implementers.

Due to this last point, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.

## Reference Implementation

Reference implementation to be created.

## Adoption Plan

To be determined.

## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community. Thanks to all persons participating in the discussions on Github and Discord to hone this improvement proposal.
