# About

This is an analysis of the algorithm used in Ethereum to assign validators to committees. Insights can be used as a reference for the design of validator assignment in NEAR. In addition, Ethereum can provide a benchmark for the work required to prove security properties of an algorithm for validator assignment.

# Validator assignment

There are *N* validators with stake *S* and they need to be assigned to validator committees.

The maximum stake per validator is 32 ETH and any amount staked above that limit is ignored ([ref](https://launchpad.ethereum.org/en/faq)). For activation as validator, a node requires a stake of at least 32 ETH. A validator might be penalized for actions like missing votes (inactivity) and malicious behavior, which reduces its staked amount. Once its stake drops below 16 ETH, it will be removed from the set of validators.

# Ethereum’s algorithm

In Ethereum validators are distributed into committees by a [random shuffle](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee). Note that the shuffling is independent of validator stakes, i.e. it is unweighted.

## Security

Ethereum operates under the assumption that at least `2/3` of validators are honest. In this case, a malicious validator controls at most `1/3` of attestations. A malicious validator *M* can control a committee if `2/3` of its seats are assigned to *M*. Hence the analysis of the security of Ethereum’s validator assignment algorithm is based on the following question: If there is a malicious validator *M* holding `1/3` of active validator stake, what is the probability of *M* controlling `2/3` of a committee? In other words, if the system as a whole is still secure (`2/3 <= honest validators && M <= 1/3`), what is the probability of *M* controlling a committee? This event is referred to as a *safety failure* and we call the probability of its occurrence *p_sf*.

For an answer of this question, the Ethereum community refers ([a](https://eth2book.info/capella/part2/building_blocks/committees/#target-committee-size), [b](https://medium.com/@chihchengliang/minimum-committee-size-explained-67047111fa20)) to a presentation held by Vitalik: He models the probability of assigning *M* to a committee with the binomial distribution with the following parameters. The probability of a “success” (assigning a malicious validator) is `1/3`. There are *n* draws which represents the committee size and the number of malicious validators assigned to the committee is *k*. A safety failure occurs if `k >= 2n/3`. Then a desired threshold for *p_sf* is set to `2^{-40}` and *n* is chosen such that `p_sf <= 2^{-40}`. The smallest committee size *n* satisfying this requirement is 111. Due to the intent of using constants which are a power of 2 in the Ethereum specification, the committee size is set to 128 (the smallest power of 2 greater than 111).

# Issues

## Model

[This article](https://medium.com/logos-network/sharding-how-many-shards-are-safe-bc361c487083) points out two issues in the modeling of the probability of a safety failure:

1. Use of the binomial distribution which assumes sampling _with_ replacement while, actually, there is no replacement. Once assigned to a committee, a validator cannot be assigned to another committee for the same slot.
2. The analysis considers only one shard and wrongly assumes the following probabilities are equal: `P(failure in shard 1) = P(at least one failure across all shards)`. To illustrate the error in that assumption, the author compares a safety failure in a single shard to the occurrence of tails in a coin flip. The more coin flips (shards), the higher the probability of at least one tails (safety failure).

The first issue can be addressed by using the hypergeometric distribution instead of the binomial distribution. However, this does not address the second issue since successive hypergeometric samples are not independent.

## Inconsistency regarding stake weights

The validator assignment algorithm and the analysis of its security implicitly assume equal stakes of validators. The shuffling function defined in the spec can be found [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee) and a corresponding implementation in the `lighthouse` consensus client is available [here](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#compute_committee). Neither takes validator stake into account. In addition, we are not aware of a security analysis similar to the one introduced above that does take validator stake into account.

Becoming a validator requires staking at least 32 ETH and any stake beyond that does not increase the effective balance ([ref](https://launchpad.ethereum.org/en/faq)). Hence it can be assumed any validator’s initial balance is 32 ETH. A validators balance can be decreased by penalties due to inactivity or malicious behavior. The balance may decrease to 16 ETH before a validator is removed from the set of active validators. Therefore it must be assumed that effective validator balances are in the range `16 ETH <= S <= 32 ETH `.

Differences in stake are relevant since validator votes are weighted by stake. This can be verified by examining the following functions in the specification and in `lighthouse`:

- specification: [`get_head`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/fork-choice.md#get_head),  [`get_weight`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/fork-choice.md#get_weight)
- `lighthouse`: [`get_head`](https://github.com/sigp/lighthouse/blob/dfcb3363c757671eb19d5f8e519b4b94ac74677a/consensus/fork_choice/src/fork_choice.rs#L479-L530), [`get_weight`](https://github.com/sigp/lighthouse/blob/dfcb3363c757671eb19d5f8e519b4b94ac74677a/consensus/proto_array/src/proto_array_fork_choice.rs#L779-L786)


If there were indeed no analysis taking different effective balances among validators into account (i.e. we didn’t just miss it), there would be a gap in the security analysis of the algorithm.

### Notes on consensus clients

On an Ethereum node, the consensus client handles consensus related logic. The Ethereum website [lists](https://ethereum.org/en/developers/docs/nodes-and-clients/#consensus-clients) several consensus client implementations. On of them is [`lighthouse`](https://lighthouse.sigmaprime.io) which is developed and maintained by Sigma Prime. With a share of [~33% of the network,](https://clientdiversity.org/#distribution) it is the second most used client after [Prysm](https://prysmaticlabs.com) with a share of ~46%. Given that multiple client implementations operate successfully, it should be sufficient to look into a single implementation for above analysis. We have chosen `lighthouse` since it is written in Rust, which is also widely used across the NEAR ecosystem.

# Other sources

- [Vitalik's presentation](https://web.archive.org/web/20190504131341/https://vitalik.ca/files/Ithaca201807_Sharding.pdf) referred to by the Ethereum community.
- [Proof of stake (POS) overview](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos)
- [Attestations](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/attestations/)
- [POS attack and defense](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/attack-and-defense)


