<pre>
  WIP: WIP-0022
  Layer: Consensus (hard fork)
  Title: Data request reward collateral ratio
  Authors: drcpu <drcpu@protonmail.com>
  Discussions-To: https://github.com/witnet/witnet-rust/discussions/2168
  Status: Draft
  Type: Standards Track
  Created: 2022-10-04
  License: BSD-2-Clause
</pre>

## Abstract

This improvement proposal aims to introduce a data request reward to collateral ratio. This ratio will be enforced using a base ratio at the protocol level, but can also be raised above this level by individual miners.

## Motivation and rationale

Node operators in the Witnet network can participate in two tasks: building blocks containing transactions and solving data requests. This improvement proposal aims to improve the operator's experience of the latter task. When node operators are eligible to solve data requests, they have to post a collateral which serves as a measure to keep them honest. In turn, if they are honest and reveal an answer to the data request which aligns with the answers of other nodes, they receive a reward. At the moment, the default collateral requirement is 1 WIT and the minimum reward for data request solvers is 1 nanoWIT. This means that a data request solver has to stake 1 billion times the reward he can earn. Some [data requests](https://www.witnet.network/search/87495de399cb4d12ce37b959149278a3a299253342deccd987a02bc5af803818) even amplify this imbalance by increasing the required collateral while keeping the reward at 1 nanoWIT resulting in the data request solver having to stake 2.5 billion times the reward they can earn. [Other requests](https://www.witnet.network/search/2c9e60dc9f8492310bb1982fd87801f4a8ef0f3c261035826256b312f836ee5c) do pay a more fair reward for the required collateral, but even there the reward to collateral the ratio is still 1 to 5000.

Due to these imbalanced reward to collateral ratios, revealing an out-of-consensus value can be extremely punishing. Observing data requests in the past six months reveals that the average `lie rate` is about 0.2% meaning the average node operator reveals an out-of-consensus value for every 500 data requests he solves. Obviously, if the out-of-consensus reveal was an attempt at manipulating a data request, this punishment is justified. However, sometimes node operators involuntarily reveal an out-of-consensus value. In some cases, out-of-consensus reveals can be explained by [badly configured data requests](https://www.witnet.network/search/aa7c0529371ccc4d09c01e0fbffdf90b1829117f141e098ec06f12a6addb9581). In other cases, like with price-feed requests which constitute the majority of data requests, quering the requested API's a fraction of a second too late yields a value which is too different from the other reveals, resulting in the node operator's collateral being slashed.

At the moment, despite the average reward to collateral ratio being quite low, revealing an out-of-consensus value is not actually punishing because slashed collateral is distributed amongst honest nodes. While a node operator will lose significantly more collateral than the potential reward when he involuntarily reveals an out-of-consensus value, the fact that he can earn the slashed collateral of other operators when he is honest offsets this.

However, redistributing slashed collateral introduces certain types of collateral-stealing attacks, and [WIP-0023](https://github.com/drcpu-github/WIPs/blob/burn-slashed-collateral/burn-slashed-collateral.md) proposes to burn slashed collateral to mitigate these attacks. Under the assumption that this WIP is accepted, solving data requests with significantly imbalanced ratios does not make economic sense anymore. The only reason node operators should participate in solving them is because it improves their odds to mine blocks and earn the associated reward. This could result in node operators aiming to solve the minimum amount of data requests possible to still profit from increased mining odds, which is obviously bad for the overall network health.

A potential downside of introducing a reward to collateral ratio at the node level is that it is possible to manipulate the reputation system by creating and solving data requests which other nodes will not solve. This proposal solves this issue by introducing this ratio at the protocol level. A data request which does not adhere to the minimum reward to collateral ratio will be considered as invalid. If a block is proposed including such a data request, it is also considered as invalid.

One additional advantage of a protocol-enforced reward to collateral ratio is that several (griefing) attacks becoming significantly less interesting from an economic viewpoint. Without a reward to collateral ratio, it is possible to create data requests that lock up collateral from other nodes or aim to introduce collateral slashing for virtually no money. For example, you could create malicious requests requiring 100 WIT collateral and set all rewards to 1 nanoWIT resulting in the data request being virtually free. If a reward to collateral ratio is introduced, this type of griefing attacks will cost a tangible amount of money to execute.

## Specification

### Introduction of a protocol-enforced data request reward collateral ratio

Properly enforcing a reward to collateral ratio at the protocol level requires adding this ratio as a consensus constant. Adding a consensus constant will change the magic number nodes used to validate communication between nodes. Therefore, proper precautions need to be implemented such that this number only changes when the WIP is accepted.

The moment the WIP is activated, nodes which receive a data request that does not adhere to the reward to collateral ratio MUST refuse to add it to their local mempool.

```Rust
let dr_tx_reward_collateral_ratio = dr_tx.body.dr_output.witness_reward / dr_tx.body.dr_output.collateral;
if dr_tx_reward_collateral_ratio < self.minimum_reward_collateral_ratio {
    // Do not insert the transaction in our local memory pool
    return vec![Transaction::DataRequest(dr_tx)];
} else {
    let weight = f64::from(dr_tx.weight());
    let priority = OrderedFloat(fee as f64 / weight);

    ...

    self.dr_transactions.insert(key, (priority, dr_tx));
}
```

Each and every received by a node is validated, which implies that all transactions in it are checked against a set of rules. If a node receives a block containing a data request that does not adhere to the required reward to collateral ratio, they should mark the block as invalid and refuse it as a candidate.

```Rust
pub fn validate_data_request_output(
    request: &DataRequestOutput,
    required_reward_collateral_ratio: u64,
) -> Result<(), TransactionError> {

    ...

    let reward_collateral_ratio = request.witness_reward / request.collateral;
    if reward_collateral_ratio < required_reward_collateral_ratio {
        return Err(TransactionError::RewardTooLow {
            reward_collateral_percentage,
            required_reward_collateral_ratio,
        });
    }

    ...

}
```

### Introduction of a configurable data request reward collateral ratio

Next to a protocol-enforced reward to collateral ratio, node operators SHOULD be able to set a ratio they consider acceptable. This ratio MUST NOT be set lower than the protocol-enforced ratio. Setting the ratio higher than the protocol-enforced ratio implies that these node operators MAY refuse to add received data requests with a lower ratio to their local mempool. However, if they receive a block with a ratio which is lower than the ratio they accept but higher than the protocol-enforced ratio, they MUST accept it as a valid candidate.

### Reward collateral ratio proposal

There are two sensible values for a protocol-enforced reward to collateral ratio.

The first one is that the ratio should offset the involuntary lie ratio. Assuming this is 0.2%, the minimum reward to collateral ratio should be 1 to 500.

The second one can be derived from the maximum amount of witnesses in a data request which, following [WIP-0007](https://github.com/witnet/WIPs/blob/master/wip-0007.md), is 125. Assuming creating data requests requiring 125 witnesses has as goal to achieve maximum security, the reward for them should be proportional. This implies that such data request should pay out a reward equal to the required collateral, or that the reward to collateral ratio should be 1 to 125.

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

A reference implementation for the proposed protocol improvement can be found in the [pull request #2229](https://github.com/witnet/witnet-rust/pull/2229) of the [witnet-rust](https://github.com/witnet/witnet-rust/) repository.

## Adoption Plan

To be determined.

## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community. Thanks to all persons participating in the discussions on Github and Discord to hone this improvement proposal. Furthermore, thanks to [Tomasz Polaczyk](https://github.com/tmpolaczyk) for contributing to the reference implementation in Rust.
