<pre>
  WIP: WIP-0023
  Layer: Consensus (hard fork)
  Title: Burn slashed collateral
  Authors: drcpu <drcpu@protonmail.com>
  Discussions-To: https://github.com/witnet/witnet-rust/discussions/2238
  Status: Draft
  Type: Standards Track
  Created: ????-??-??
  License: BSD-2-Clause
</pre>

## Abstract

This improvement proposal introduces the idea of burning slashed collateral to prevent malicious node operators from profiting of collateral-stealing attacks.

## Motivation and rationale

Succesfully solving data requests happens in four different stages. First the data request is selected from the memory pool and included in a block. Now it moves to the commitment stage where nodes calculate whether they are eligible to solve it. When they are eligibile, they will solve the data request by quering all requested API's. After enough commit transactions have been broadcasted, the data request moves to the reveal stage. Here the nodes that broadcasted a commitment transaction will broadcast the associated reveal. After the network has either seen enough reveals, the data request will move to the tally stage. Here all reveals will be analyzed following the rules defined by the data request. Nodes which reveal an in-consensus value are rewarded, nodes which reveal an error receive their collateral back without a reward and nodes which reveal an out-of-consensus value are punished by the network confiscating the collateral they put up. This proposal focuses on the nodes that reveal an out-of-consensus variable.

Confiscated collateral is currently redistributed to the nodes that solved a data request honestly. The problem with that approach is that it creates an obvious incentive for malicious nodes operators to perform several types of collateral-stealing attacks on other witnesses. Two types of attacks are described in [the discussion](burn-slashed-collateral) that lead to this WIP.

The first and easiest to execute attack revolves around creating a data request with 'inconsistent' sources. This goal here is for a number of witnesses to reveal an out-of-consensus value such that the quorum requirements are still satisifed. This results in a number of coins (`liars * collateral`) being slashed and redistributed amongst honest nodes. The attacker does not directly benefit from creating this specific data request because he has no control over which nodes earn the redistributed collateral. However, if he repeats this attack often enough and controls a non-negligible part of the ARS his nodes will statistically gain more Witnet coins than if the attack was not taking place due to his nodes receiving part of the slashed coins.

The second attack is a bit more involved. Instead of targetting random nodes with a malicious data request, the attacker first builds a mapping of public keys to IP addresses. Once nodes have been identified he waits until his node is selected to solve a data request. At the reveal stage of the data request, he will then launch a DDoS attack on some of the other witnessing nodes such that the data request still meets the quorum demands. This will result in the attacked nodes being unable to reveal their answer and they will get slashed. As the attacker was also chosen to solve this data request, he directly benefits from this data request since he will receive part of the slashed collateral.

The best solution to mitigate this type of attacks is to render them economically uninteresting. This can be achieved through burning the slashed collateral instead of redistributing it amongst honest nodes. Of course it is still possible to execute these attacks and purposely burn collateral, but by taking away the economic incentive, they are less likely to occur.

## Specification

Instead of distributing slashed collateral amongst all honest revealers, the slashed collateral is sent to a burn address. The new `calculate_witness_reward` function needs to be rewritten such that instead of a slashed collateral remainder, it returns the witness reward and total amount of burned collateral:

```
pub fn calculate_witness_reward(
    commits_count: usize,
    // Number of values that are out of consensus plus non-revealers
    liars_count: usize,
    // To calculate the reward, we consider errors_count as the number of errors that are
    // out of consensus, it means, that they do not deserve any reward
    errors_count: usize,
    reward: u64,
    collateral: u64,
) -> (u64, u64) {
    let honests_count = (commits_count - liars_count - errors_count) as u64;

    if commits_count == 0 {
        (0, 0)
    } else if honests_count == 0 {
        (collateral, 0)
    } else {
        let burned_collateral = collateral * (liars_count as u64);

        (
            reward + collateral,
            burned_collateral,
        )
    }
}
```

Furthermore, the `create_tally` function will need to add an extra output to the transaction if it contains any liars. This output could take similar form as the `change_output`:

```
let (reward, burned) = calculate_witness_reward(
    commits_count,
    liars_count + non_reveals_count,
    errors_count,
    dr_output.witness_reward,
    collateral,
)

...

if liars_count > 0 {
    let vt_output_burn = ValueTransferOutput {
        wit10000000000000burnbabyburn0000000000000,
        value: burned,
        time_lock: 0,
    };
    outputs.push(vt_output_burn);
}

...

```

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

A reference implementation for the proposed protocol improvement can be found in the [pull request #2229](https://github.com/witnet/witnet-rust/pull/2229) of the [witnet-rust] repository.

## Adoption Plan

An activation date is proposed for <Month> <Day> <Year> at 9am UTC, that is, protocol epoch #<Epoch>.

The signaling bit for this improvement proposal is `4` (`0b00000000000000000000000000010000`) and the `first_signaling_epoch` is set to #<Epoch>.

## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community. sThanks to all persons participating in the discussions on Github and Discord to hone this improvement proposal.

[witnet-rust]: https://github.com/witnet/witnet-rust/
[burn-slashed-collateral]: https://github.com/witnet/witnet-rust/discussions/2238
