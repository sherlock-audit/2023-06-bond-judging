Yuki

medium

# The function claimNextEpochRewards can't be used to move through epochs one txn at a time as it doesn't update the mapping lastEpochClaimed.

## Summary
The function claimNextEpochRewards can't be used to move through epochs one txn at a time as it doesn't update the mapping lastEpochClaimed.

## Vulnerability Detail
The function claimNextEpochRewards() is supposed to claim all outstanding rewards for the user on their next unclaimed epoch. And to allow moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

However this function doesn't work as intended and will be useless as it won't update the lastEpochClaimed to the next epoch in the line.

Imagine the following scenario:

1. User unstakes all of his funds and claims his rewards at epoch 10, his lastEpochClaimed will equal 10.
2. The user returns and stakes again at epoch 20, after the stake his rewardsPerTokenClaimed will equal the current rewardsPerTokenStored.
3. He calls the function claimNextEpochRewards in order to avoid gas issues.
4. In the function the else statement will be triggered as userClaimedRewardsPerToken > rewardsPerTokenEnd, so the function will try to claim rewards for the next epoch in line.
5. The problem is that the function _claimEpochRewards will return 0 before updating the user's lastEpochClaimed to the next epoch in line, as the if statement (userRewardsClaimed >= rewardsPerTokenEnd) will be triggered.

After the execution of the function claimNextEpochRewards, nothing changed and the lastEpochClaimed wasn't updated, which means that the user can't use the function to move through the epochs to avoid gas issues.

```solidity
    /// @notice Claim all outstanding rewards for the user for the next unclaimed epoch (and any remaining rewards from the previously claimed epoch)
    function claimNextEpochRewards()
        external
        nonReentrant
        updateRewards
        tryNewEpoch
        returns (uint256)
    {
        // Claims all outstanding rewards for the user on their next unclaimed epoch. Allows moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

        // Revert if user has no stake
        if (stakeBalance[msg.sender] == 0) revert OTLM_ZeroBalance();

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

        // If the last epoch claimed is equal to the current epoch, then try to claim for the current epoch
        if (userLastEpoch == epoch) return _claimEpochRewards(epoch);

        // If not, then the user has not claimed rewards from the next epoch
        // Check if the user has claimed all rewards from the last epoch first
        uint256 userClaimedRewardsPerToken = rewardsPerTokenClaimed[msg.sender];
        uint256 rewardsPerTokenEnd = epochRewardsPerTokenStart[userLastEpoch + 1];
        if (userClaimedRewardsPerToken < rewardsPerTokenEnd) {
            // If not, then claim the remaining rewards from the last epoch
            uint256 remainingLastEpochRewards = _claimEpochRewards(userLastEpoch);
            uint256 nextEpochRewards = _claimEpochRewards(userLastEpoch + 1);
            return remainingLastEpochRewards + nextEpochRewards;
        } else {
            // If so, then claim the rewards from the next epoch
            return _claimEpochRewards(userLastEpoch + 1);
        }
    }
```

## Impact
Duo to logic error, the function claimNextEpochRewards will be unusable when wanting to move through epochs one txn at a time, as in some scenarios it doesn't update the mapping lastEpochClaimed. 

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L430

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L485

## Tool used

Manual Review

## Recommendation
The solution for this problem is simple, you can update the lastEpochClaimed to the next epoch in line, if the function _claimEpochRewards returns 0.

The function _claimEpochRewards returning zero means that either of the if statements were trigger:

- if (userRewardsClaimed >= rewardsPerTokenEnd) return 0;
- if (uint256(optionToken.expiry()) < block.timestamp) return 0;

If option was expired the lastEpochClaimed would already be updated, but it doesn't if its updated twice since the value is the same.

```solidity
    function claimNextEpochRewards()
        external
        nonReentrant
        updateRewards
        tryNewEpoch
        returns (uint256)
    {
        // Claims all outstanding rewards for the user on their next unclaimed epoch. Allows moving through epochs one txn at a time if desired or to avoid gas issues if a large number of epochs have passed.

        // Revert if user has no stake
        if (stakeBalance[msg.sender] == 0) revert OTLM_ZeroBalance();

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

        // If the last epoch claimed is equal to the current epoch, then try to claim for the current epoch
        if (userLastEpoch == epoch) return _claimEpochRewards(epoch);

        // If not, then the user has not claimed rewards from the next epoch
        // Check if the user has claimed all rewards from the last epoch first
        uint256 userClaimedRewardsPerToken = rewardsPerTokenClaimed[msg.sender];
        uint256 rewardsPerTokenEnd = epochRewardsPerTokenStart[userLastEpoch + 1];
        if (userClaimedRewardsPerToken < rewardsPerTokenEnd) {
            // If not, then claim the remaining rewards from the last epoch
            uint256 remainingLastEpochRewards = _claimEpochRewards(userLastEpoch);
            uint256 nextEpochRewards = _claimEpochRewards(userLastEpoch + 1);
            return remainingLastEpochRewards + nextEpochRewards;
        } else {
            // If so, then claim the rewards from the next epoch
            uint256 rewards = _claimEpochRewards(userLastEpoch + 1);
            if (rewards == 0) lastEpochClaimed[msg.sender] = userLastEpoch + 1;
            return rewards;
        }
    }
```