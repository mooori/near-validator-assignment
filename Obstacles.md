# About

This document discusses some issues that might complicate the rollout of a new validator assignment algorithm for Near. These issues are _not_ considered deal breakers.

# Calculation of rewards and slashing amounts

As of now, both a validator's rewards and slashing amounts are proportional to its fraction of total stake. This is appropriate for the current stage of sharding in which chunk producers are expected to be assigned to a [single chunk only](https://nomicon.io/ChainSpec/SelectingBlockProducers#steps-1).

In the new [proposed algorithm](https://docs.google.com/document/d/1C-w4FNeXl8ZMd_Z_YxOf30XA1JM6eMDp5Nf3N-zzNWU/edit#heading=h.7wdbrn8ypjiw) for validator assignment with stateless validation, validators with large stake are expected to be assigned multiple seats pertaining to more than one shard. In this case, a validator might earn a reward for some seats but not for others. For instance due to hardware limitations or a bug in `nearcore`. If that happens, a reward in proportion to the validator's total stake would not be adequate. Instead, rewards should be proportional to the stake assigned to seats for which the validator performed its duties.

Similar considerations apply to the calculation of slashing amounts, which should be tied to seats for which the validator failed to perform its duties rather than to the validator's total stake.

## Delegated staking

@birchmd pointed out that above issue also requires adjustments in the calculation of delegator rewards in [delegated staking](https://github.com/near/core-contracts/tree/master/staking-pool). Two possible approaches outlined by @birchmd are:

1. Tying a delegator's rewards to its proportion of the validator's rewards.
2. Tying a delegator's stake to particular seats.

## Security considerations

If the calculation of rewards does not account for seats whose duties were not performed, a malicious validator might find ways to receive rewards that it should not have earned.
