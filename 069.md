BenRai

high

# There are more options issued for an epoch than should be issued based on the epoch length

## Summary

Depending on the time the next epoch is started, the amount of issued optionTokens for the current epoch can be significantly higher than to be expected.

## Vulnerability Detail

If the `epochDuration` is 10 days and the `rewardRate` is 100 options a day one would expect that only 10 * 100 = 1000 option Tokens will be issued for the epoch. But since the amount of `optionTokens` given as a reward for the epoch are determined by the `epochRewardsPerTokenStart[]` of the current epoch and the `epochRewardsPerTokenStart[]` of the following epoch, the actual amount of `optionTokens` given as a reward for the epoch can be higher than the set `epochDuration` would suggest. This depends on when `tryNewEpoch()` is called. 

This can lead to significant lose of funds for the OTLM owner/receiver who provides the payout token that are exchanged for the option token.

Example:
For simple calculations:
rewardsPerTokenStored = 0;
totalBalance = 100 (and does not change in this example)


To reward the stakes of the `stakedTokens`, the owner of a OTLM wants to issue 100 options for 10 days with a very low strike price. Therefore, he sets `epochDuration` to 10 days, `rewardRate` to 100 and the strike price to 1 (An attractive price for the option based on the current market price of the `payoutToken` would be 100).  


He starts epoch0 setting its `epochRewardsPerTokenStart[]` to 0. He also changes the strike price to 100 to make sure that after the 10*100 = 1000 `optionTokens` with the low strike price are issued, the strike price for `optionTokens` in epoch1 is 100 and the options bring in revenue to the `reciever` again. 

For simplicity purposes we assume that during the epoch no one is triggering `updateRewards`.
The 10 days go buy but the only after the 11th day a function is called that triggers the `updateRewards`, and the `tryNewEpoch` modifier. 

`updateRewards` triggers the following calculations:

 `rewardsToApply = ((block.timestamp - lastRewardUpdate) * rewardRate) / REWARD_PERIOD;` 
=>  rewardsToApply = 11 days * 100 / 1 day = 1100

rewardsPerTokenStored = rewardsPerTokenStored + (rewardsToApply * 10 ** stakedTokenDecimals) / totalBalance;
=> rewardsPerTokenStored = 0 + 1100 / 100 = 11

When the new epoch is triggered `rewardsPerTokenStored` (11) is set as `epochRewardsPerTokenStart` for epoch1. 

If everyone claims their rewards for epoch0 this would be the following number of `optionTokens`:

(11-0)*100 = 1100 



This means, that for epoch0 1100 `optionTokens` with a strike price of 1 can be claimed.
If the 100 additional options with a `strikePrice` of 1 are executed, the owner of the OTLM misses out on (100-1)*100 = 9900 of revenue of the `quoteToken`.   


## Impact

Since more options are minted for epoch0 than planed, the owner misses out on revenue if the options are executed. 


## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L175-L182

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L185-L196

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L514-L534

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L475-L478


## Tool used

Manual Review

## Recommendation

Instead of using the `epochRewardsPerTokenStart` of the following epoch to determine how many `optionTokens` should be minted, the ` epochRewardsPerTokenEnd` should be tracked.
 
-  epochRewardsPerTokenStart for epoch 0 is 0
- epochRewardsPerTokenStart of all following epochs is the ` epochRewardsPerTokenEnd ` of the previous epoch
- when starting a new epoch `epochRewardsPerTokenEnd` of the last epoch needs to be calculated base on `epochStart`, `epochDuration`, `lastRewardUpdate` and `rewardRate` 
