<pre>
    WIP: WIP-mariocao-blockweight
    Layer: Consensus (hard fork)
    Title: Block weight
    Authors: Mario Cao <mario@stampery.co>
             Gorka Irazoqui <gorka@stampery.co>
    Discussions-To: `#dev-general` channel on Witnet Comunnity's Discord server
    Status: Draft
    Type: Standards Track
    Created: 2020-06-10 
    License: BSD-2-Clause
</pre>


## Abstract

This WIP discusses the maximum block size in terms of weighting transactions differently in relation to their type and structure. 

This proposal seeks to guarantee a sustainable block chain growth while proactively keeping a healthy system performance in terms of workload and data structures maintained by Witnet nodes.


## Motivation and Rationale

Memory is a scarce resource that plays a key role in blockchain design being in charge of not only storing blocks anchored to the chain (Disk) but also serving the data that the software needs to perform computations (RAM). Ensuring the healthiness of such hardware pieces implies improving the liveness of the chain itself. Not in vain, if the data to be stored/treated exceeds the hardware capacity it would prevent miners from contributing successfully to the chain.

The aforementioned scarcity necessarily implies that data need to have certain limits when being treated in a decentralized network. The current implementation of the Witnet protocol does not impose these limits: a miner with big storage capabilities can propose blocks that are big enough to saturate the smaller storage capabilities of the rest of the nodes. It goes without saying that this is a clearly undesirable feature that should be prevented. 

In addition, the current Witnet implementation does not make any distinction between data request-related transactions and Value Transfer transactions. This implies that, if the fee that VTT is high enough VTTs could end up delaying the execution of data requests, defeating the main purpose of the Witnet network. 

Finally, the current absence of data size limits could damage the efficiency of the network even if they do not trigger an exhaustion of hardware resources. As an example, an attacker could reduce significantly the performance of the network by creating a big amount of dust in the form of Unspent Transaction Outputs (UTXOs). Or even worse, it can introduce a big amount of data requests with little to zero reward aiming at benefiting of reputation minting in a short period of time.

This proposal aims at solving the aforementioned issues by limiting the rate at which the chain and data request pool grow. In the following sections current transactions sizes and target data growth rates will be analyzed, ending up with a specification of weights that should be assigned to different transaction types to make data storage sustainable.


## Proposal

This proposal aligns different set of goals:

- Sets a block size limit
- Avoids fee competition between different transaction types
- Favors data requests in detriment of value transfer transactions
- Proactively models the system workload throughput
- Subsidizes UTXO join transactions in detriment of UTXO split transactions

The proposal is based on the combination of:

- reserved block space for different types of transactions by using *weights units*, and
- transactions are weighted depending on their type, characteristics and impact in the system (e.g. computation, memory allocation, network bandwidth).

*Weight unit* is a measurement used to compare different transactions types and their content. Transaction weights are therefore derived based on actual sizes (in bytes) but they might also consider additional factors. Therefore a transaction weight should not be confused with current size as in most cases it will differ.

It is assumed that no data request scheduling mechanism is included in the Witnet protocol as it would include several attack vectors (see [Future Work](##-Future-Work)).


### Block weight

Block space for transactions will be splitted with regards to the transaction type. The main goal is to avoid spurious fee competitions between transactions of different nature. This is the case of value transfer transactions and data requests.

Block space for transactions will be allocated by using compartments called buckets. These buckets could be bounded or unbounded depending on the transaction type.

```text
+-------------+--------------+---------------+
|             |              |    Mint       |
|    Value    |     Data     |    Commits    |
|  Transfers  |   Requests   |    Reveals    |
|             |              |    Tallies    |
+-------------+--------------+---------------+
   Bounded        Bounded        Unbounded
```

Value transfer and data request transactions will be allocated by filling their respective buckets. The main goal is to isolate both fee markets so that it would be impossible to influence one market from the other.

Additionally, in order to favor data requests over value transfer transactions, the data request bucket should be significantly bigger than the value transfer bucket.

The rest of transactions types can be allocated in an unbounded compartment because they are already limited by the previous accepted data requests. In other words, the amount of commits, reveals and tally transactions will be always dependent on the previously accepted and non-resolved data request transactions. In the case of mint transactions, there can only be a single transaction per block.

Defining the size of the two first buckets will have a direct consequence on the overall block chain size across time (remember that the third is indirectly limited by the data request bucket). Therefore they should be carefully defined in order to allow a sustainable block chain growth.


### Transaction weights

The transaction weights aim to take into account future workload that might be derived from the transactions themselves. For example, in the case of data request transaction, they will lead to a set of commit, reveal and tally transactions, which will have a direct impact in the system workload and performance.

Bearing this in mind, the main goals of the transaction weights are twofold:

1. Proactively shape the overall system workload in terms of the data request lifecycle (i.e. for each data request, many commits and reveals should follow).
2. Proactively contribute to the healthiness of data structures stored in the node's storage (ram and disk). This is the case of favoring UTXO join operations instead of split operations to decrease the UTXO set stored on each node.

The sum of all transaction weights within a block MUST BE AT MOST the block weight defined by the bounded buckets.

```
SUM(transaction_weights) <= BUCKET1_LIMIT + BUCKET2_LIMIT
```


#### Data requests

The Data Request transaction formally defines the number of commit, reveal and tally transactions that will included in the subsequent blocks as well as the computation effort that each witness node will have to devote. 

The data request weight should be influenced by:

- number of commits: always limited in sized
- number of reveals: size depends on the data request return type
- tally: length depends on return type and slightly on the number of witnesses
- data request complexity: refers to the associated computational and storage complexity that the data request will trigger across the system. It highly depends on:
  - Number of sources
  - RADON scripts (operations to retrieve and aggregate data)
  - Return types

In terms of storage, data request weights will consider **in advance** all subsequent transaction sizes (commits, reveals and tally). With respect to computational cost, data request weights will include a multiplicative factor dependent on the complexity defined by the data request.

The following equation aims to model all aforementioned factors:

```
DR_weight = DR_size*alpha + W*COMMIT + W*REVEAL*beta + TALLY*beta + W*OUTPUT_SIZE
```

where:
 - **DR_weight** is the weight applied to a data request transaction
 - **DR_size** is the total data request length
 - **COMMIT** is the weight unit assigned to commit transactions
 - **REVEAL** is the weight unit assigned to reveal transactions
 - **TALLY** is the weight unit assigned to tally transactions
 - **OUTPUT_SIZE** is the size of each output, fixed in size
 - **W** is the number of witnesses specified by the data request
 - **alpha** is the data request complexity factor
 - **beta** is the return type factor


#### Value transfers

The main goal of value transfers is to keep in line the storage of nodes in terms of UXTOs. Therefore, value transfers that join several UTXOs (decreases storage footprint) are favored in detriment of value transfer that split to many UTXOs (increases storage footprint).

The proposed approach is weight significantly more the transaction outputs than the consumed inputs:

`VT_weight = N*INPUT_SIZE + M*OUTPUT_SIZE*gamma`

where:
 - **VT_weight** is the weight applied to a value transfer transaction
 - **gamma** is the output multiplicative factor
 - **INPUT_SIZE** is the size of each input, which is fixed in size
 - **OUTPUT_SIZE** is the size of each output, fixed in size
 - **N** is the number of inputs
 - **M** is the number of outputs


#### Rest of transactions

As the rest of transaction will be allocated in the unbounded bucket as there is no need to weight them.

Commit, reveal and tally transactions will be indirectly limited by previously accepted data requests.

Mint transactions are included by the miners are they are limited to 1 per block.

Additionally, by protocol design, commit and reveal transactions cannot be used as mechanism for transferring value as their output addresses must be equal to the node's identity.


#### Estimation of transaction lengths

In order to better understand the impact of assigning different weights to different transaction types it is important to analyze the approximate length of each kind of transaction. While some of them have fixed sizes (e.g., Mint transactions) others may depend on e.g., the number inputs/outputs. The table summarizes both the minimum size of each of these transactions and the average size under common number of inputs/outputs.

- estimation of field lengths
- do not take into consideration serialization overhead

| Transaction Type |  Minimum size  |       Total size  |  Measurements  |
| ---------------- | -------------- | ----------------- | -------------- |
| Mint             |       40 bytes |          76 bytes |    ~  40 bytes |
| Value Transfer   |      133 bytes |  `N*36+M*36+N*97` |    ~ 205 bytes |
| Data Request     | 133+data bytes |   `36+N*133+data` |    ~ 400 bytes |
| Commit           |      376 bytes |       `279+N*133` | ~ W\*412 bytes |
| Reveal           |      149 bytes |      `149+result` | ~ W\*202 bytes |
| Tally            | 124+data bytes |  `52+W*36+result` |    ~ 105+W\*36 |

where:
 - *N* refers to the number of inputs,
 - *M* to the number of outputs, 
 - *W* to the number of witnesses specified by the data request,
 - *data* refers to length of the RADON script to be executed by witnesses,
 - *result* refers to the length of the bytes retrieved by witnesses.


### Specification

Based on the proposal, the following parameter definitions are suggested:

- A maximum block weight of 100K
- A bounded VTT bucket of 20K
- A bounded Data Request bucket of 80K
- A *COMMIT_WEIGHT* of 400
- A *REVEAL_WEIGHT* of 200
- A *TALLY_WEIGHT* of 100
- *beta* is equal to 1
- *alpha* is equal to 1
- *gamma* is equal to 10

Currently the known sizes for inputs are outputs are:

- INPUT_SIZE is 133 bytes (97+36)
- OUTPUT_SIZE is 36 bytes


#### System limits

Based on the previously defined parameters, the system will be shaped in different ways:

 - block chain growth
 - value transfer throughput
 - data request throughput
 

**Blockchain growth**. A maximum block weight of 100K will lead to a theoretical yearly block chain growth of 70 G weight units. It figure does not necessarily imply a maximum block size in bytes, but it imposes a hard limit.

```
Blockchain growth = 365.25 days * 24 hours * 60 minutes * 60 seconds / 45 seconds * 100K /block = 70 G / year
```

**Value transfer throughput**. Three different scenarios can be considered in order to understand the value transfer throughput per block.

 - Splitting transfers: the maximum UTXO splitting factor by a single transaction is 55
     ```
     20 KB = 1*INPUT_SIZE + M*OUTPUT_SIZE => M = 55
     ```
 - Joining transfers: the maximum UTXO joining by a single transaction is 147
     ```
     20 KB = N*INPUT_SIZE + 1*OUTPUT_SIZE => N = 147
     ```
 - 2-2 value transfers: the maximum number of 2-2 VTTs is 20
     ```
     20 KB = K * (2*INPUT_SIZE + 2*OUTPUT_SIZE) => K = 20
     ```

** Data requests throughput**. Different set scenarios have been considered in order to understand the transaction weight and its impact in the throughput. The data request size has been approximated to 400 bytes.

 - Maximum committee requestable by a single 400 bytes Data Request is 125
     ```
     80 KB = 100 + W*636 + DR_actual_weight(~400 bytes) => W = 125
     ```
 - Number of data request with W=20 within a block is 6
     ```
     80 KB = (100 + 20*636 + DR_actual_weight(~400 bytes)) * K => K = 6
     ```
 - Number of data request with W=10 within a block is 11
     ```
     80 KB = (100 + 10*636 + DR_actual_weight(~400 bytes)) * K => K = 11
     ```
 - Number of data request with W=5 within a block is 21
     ```
     80 KB = (100 + 5*636 + DR_actual_weight(~400 bytes)) * K => K = 21
     ```
 - Number of data request with W=2 within a block is 45
     ```
     80 KB = (100 + 2*636 + DR_actual_weight(~400 bytes)) * K => K = 45
     ```
 - Number of data request with W=1 within a block is 70
     ```
     80 KB = (100 + 636 + DR_actual_weight(~400 bytes)) * K => K = 70
     ```


## Future work

The following items should be considered as future research work that complement this proposal.

Currently, the Witnet network allows data requests to be scheduled ahead in time. Discussions triggered by this proposal uncovered the current potential harms that could be exploited from a malicious usage of the data request parameter `not_before`. For example, an attacker may create a vast number of data requests to be executed at a specific point in time, thus generating unhandleable block size that cannot be processed in time during an epoch by Witnet nodes.

The design of the *alpha* parameter. This parameters has been considered equal to 1 in this proposal, but should be carefully designed taking into account the associated computational cost given the number of sources and RADON scripts.

The design of the *beta* parameter. This parameter has been considered equal to 1 in this proposal, but it should reflect the additional storage cost that different return types may trigger. Currently there is no clear limitation enforced by the protocol on the number of bytes to be retrieved by witnesses.

There are clear potential storage savings if a checkpointing/pruning mechanism in implemented in the Witnet protocol, e.g. triggered every period of time in the order of months. Old data requests transactions (e.g. commits and reveals) do not have necessarily an added value to the protocol after a certain time has passed.


## Adoption Plan

Due to the undeniable impact on consensus of this proposal, it becomes apparent that these protocol changes need to be introduced through a coordinated upgrade of the whole network.

This proposal mandates the application of distinct validation rules for transactions and blocks that predate its eventual activation. In consequence, the protocol changes proposed herein are expected to be applied on a newly bootstrapped block chain in which the updated protocol must be applied starting from the genesis block itself.


## Acknowledgements

This proposal has been cooperatively devised by many individuals from the Witnet development community.

[WIP-0006]: https://github.com/witnet/WIPs/blob/master/wip-0006.md