# [M-1] StakingPoolBase#migrate - totalPrincipal not updated leading to RewardVault reading wrong values 
https://github.com/code-423n4/2023-08-chainlink-findings/issues/955 
## Lines of code
https://github.com/code-423n4/2023-08-chainlink/blob/e0cfce3073a3c9443eac55a377ff74f22e23c163/src/pools/StakingPoolBase.sol#L230-L253

## Vulnerability details
## Impact
When a user calls migrate on a staking pool, it does not update the total principal value of the pool. This results in wrong values being supplied to reward vault resulting in wrong reward amounts.

## Proof of Concept
Both StakingPoolBase#onTokenTransfer and StakingPoolBase#unstake updates the total principal value stored in the staking pool in some way like the following:

```javascript
uint256 claimedReward = s_rewardVault.finalizeReward({
      staker: msg.sender,
      oldPrincipal: stakerPrincipal,
      unstakedAmount: amount,
      shouldClaim: shouldClaimReward,
      stakedAt: stakedAt
    });

    s_pool.state.totalPrincipal -= amount;
```
However, in `StakingPoolBase#migrate`, `s_pool.state.totalPrincipal` is not updated. This would result in a wrong value being supplied to RewardVault when the vault calls the `getTotalPrincipal` function on the staking pool. As a result, values calculated by the reward vault becomes faulty.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Update `s_pool.state.totalPrincipal` when a user migrates to another pool.

## Assessed type
Other

