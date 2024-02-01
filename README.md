# WIPs
__Witnet Improvement Proposals__ (WIPs) are documents proposing, specifying and standardizing new protocol features, processes and methods in the Witnet community.

People wishing to submit WIPs, first should propose their idea or document to the [Witnet community Discord][discord]. After discussion, please open a Pull Request to this repository. After copy-editing and acceptance, it will be published here.

This repository is maintained by Witnet Foundation. We are fairly liberal with approving WIPs, and try not to be too involved in decision making on behalf of the community. The exception would be the very rare cases of dispute resolution when a decision is contentious and cannot be agreed upon. In those cases, the conservative option will always be preferred.

Having a WIP here does not make it a formally accepted standard until its status becomes Final or Active.

| Number | Layer | Title | Owner | Type | Status |
|:------:|:-----:|:-----------:|:----------------------------:|:-------:|:--------:|
| [1](wip-0001.md) |  | WIP process | [Adán SDPC](https://github.com/aesedepece) | Process | Final |
| [2](wip-0002.md) | Consensus (hard fork) | Collateralization of Witnessing Activity | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [3](wip-0003.md) | Consensus (hard fork) | WIT issuance schedule | [Daniele Levi](https://github.com/burguesia) | Standards Track | Final |
| [4](wip-0004.md) | Consensus (hard fork) | BLS signature propagation and aggregation | [Gorka Irazoqui](https://github.com/girazoki) and [Claudia Bartoli](https://github.com/clbartoli) | Standards Track | Proposed |
| [5](wip-0005.md) | Consensus (soft fork) | ARS merkelization | [Gorka Irazoqui](https://github.com/girazoki) | Standards Track | Proposed |
| [6](wip-0006.md) | Consensus (hard fork) | Coexistence of BLS and Secp256k1 keys | [Gorka Irazoqui](https://github.com/girazoki) | Standards Track | Proposed |
| [7](wip-0007.md) | Consensus (hard fork) | Block weight | [Mario Cao](https://github.com/mariocao) and [Gorka Irazoqui](https://github.com/girazoki) | Standards Track | Final |
| [8](wip-0008.md) | Consensus (hard fork) | Limits on data request concurrency | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [9](wip-0009.md) | Consensus (hard fork) | Adjust mining probability | [Mario Cao](https://github.com/mariocao) and [Gorka Irazoqui](https://github.com/girazoki) | Standards Track | Final |
| [10](wip-0010.md) |  | Feb 2021 Chain Fork Post-Mortem | [The witnet-rust developers](https://github.com/witnet/witnet-rust/graphs/contributors) | Informational | Final |
| [11](wip-0011.md) | Consensus (hard fork) | Improve consistency and availability of superblock voting protocol | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [12](wip-0012.md) | Consensus (hard fork) | Set minimum mining difficulty  | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [13](wip-0013.md) |  | Apr 2021 Chain Fork Post-Mortem | [The witnet-rust developers](https://github.com/witnet/witnet-rust/graphs/contributors) | Informational | Final |
| [14](wip-0014.md) | Consensus (hard fork) | Threshold Activation of Protocol Improvements  | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [15](wip-0015.md) | Consensus (hard fork) | Amendment to WIP-0007  | [Mario Cao](https://github.com/mariocao) | Standards Track | Final |
| [16](wip-0016.md) | Consensus (hard fork) | Set minimum data request mining difficulty  | [Mario Cao](https://github.com/mariocao) | Standards Track | Final |
| [17](wip-0017.md) | Consensus (hard fork) | Add median to RADON reducers | [Mario Cao](https://github.com/mariocao) | Standards Track | Final |
| [18](wip-0018.md) | Consensus (hard fork) | Remove message argument from UnhandledIntercept RADON error | [Tomasz Polaczyk](https://github.com/tmpolaczyk) | Standards Track | Final |
| [19](wip-0019.md) | Consensus (hard fork) | Random Number Generation (RNG) functionality | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [20](wip-0020.md) | Consensus (hard fork) | Support HTTP-POST in RADON | [Tomasz Polaczyk](https://github.com/tmpolaczyk) | Standards Track | Final |
| [21](wip-0021.md) | Consensus (hard fork) | Add StringParseXMLMap to RADON operators | [Luis Rubio](https://github.com/lrubiorod) | Standards Track | Final |
| [22](wip-0022.md) | Consensus (hard fork) | Data request reward collateral ratio | [drcpu](https://github.com/drcpu-github) | Standards Track | Final |
| [23](wip-0023.md) | Consensus (hard fork) | Burn slashed collateral | [drcpu](https://github.com/drcpu-github) | Standards Track | Final |
| [24](wip-0024.md) | Consensus (hard fork) | Improved processing of numbers in oracle queries | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [25](wip-0025.md) | Consensus (hard fork) | HTTP redirects in retrievals | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [26](wip-0026.md) | Consensus (hard fork) | Introduce new EncodeReveal RADON error | [Tomasz Polaczyk](https://github.com/tmpolaczyk) | Standards Track | Final |
| [27](wip-0027.md) | Consensus (hard fork) | Increase the age requirement for using transaction outputs as collateral | [Adán SDPC](https://github.com/aesedepece) | Standards Track | Final |
| [28](draft-guidiaz-extened-radon-errors.md) | Consensus (hard fork) | Extended rules for improved handling of RADON errors in smart contracts | [Guillermo Díaz](https://github.com/guidiaz) | Standards Track | Draft |

## TAPI signals

These are the bits currently being used for signaling support for and activating Witnet Improvement Proposals through
the procedure set forth by [WIP-0014]: _Threshold Activation of Protocol Improvements (TAPI)_.

| Bit position | WIP(s)                                                  | First signaling epoch |    State    |
|:------------:|:-------------------------------------------------------:|:---------------------:|:-----------:|
| 0            | [14](wip-0014.md), [16](wip-0016.md)                    | `522240`              | `In force`  |
| 1            | [17](wip-0017.md), [18](wip-0018.md), [19](wip-0019.md) | `656640`              | `In force`  |
| 2            | [20](wip-0020.md), [21](wip-0021.md)                    | `1032960`             | `In force`  |
| 3            | [22](wip-0022.md)                                       | `1655120`             | `In force`  |
| 4            | [23](wip-0023.md)                                       | `1655120`             | `In force` |
| 5            | [24](wip-0024.md)                                       | `1655120`             | `In force` |
| 6            | [25](wip-0025.md)                                       | `1655120`             | `In force` |
| 7            | [26](wip-0026.md)                                       | `1655120`             | `In force` |
| 8            | [27](wip-0027.md)                                       | `1655120`             | `In force` |

[discord]: https://discord.gg/X4uurfP
[WIP-0014]: wip-0014.md
