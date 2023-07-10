bin2chen

medium

# claimRewards() If a rewards is too small, it may block other epochs

## Summary
When `claimRewards()`, if some `rewards` is too small after being round down to 0
If `payoutToken` does not support transferring 0, it will block the subsequent epochs
## Vulnerability Detail
The current formula for calculating rewards per cycle is as follows.

```solidity
    function _claimEpochRewards(uint48 epoch_) internal returns (uint256) {
...
@>      uint256 rewards = ((rewardsPerTokenEnd - userRewardsClaimed) * stakeBalance[msg.sender]) /
            10 ** stakedTokenDecimals;
        // Mint the option token on the teller
        // This transfers the reward amount of payout tokens to the option teller in exchange for the amount of option tokens
        payoutToken.approve(address(optionTeller), rewards);
        optionTeller.create(optionToken, rewards);
```

Calculate `rewards` formula : `uint256 rewards = ((rewardsPerTokenEnd - userRewardsClaimed) * stakeBalance[msg.sender]) /10 ** stakedTokenDecimals;`

When `rewardsPerTokenEnd` is very close to `userRewardsClaimed`, `rewards` is likely to be round downs to 0
Some tokens do not support transfer(amount=0)
This will revert and lead to can't claims

## Impact
Stuck `claimRewards()` when the rewards of an epoch is 0
## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L499
## Tool used

Manual Review

## Recommendation
```solidity
    function _claimEpochRewards(uint48 epoch_) internal returns (uint256) {
.....

        uint256 rewards = ((rewardsPerTokenEnd - userRewardsClaimed) * stakeBalance[msg.sender]) /
            10 ** stakedTokenDecimals;
+      if (rewards == 0 ) return 0;
        // Mint the option token on the teller
        // This transfers the reward amount of payout tokens to the option teller in exchange for the amount of option tokens
        payoutToken.approve(address(optionTeller), rewards);
        optionTeller.create(optionToken, rewards);
```