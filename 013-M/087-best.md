ctf_sec

medium

# Division before multiplication result in loss of token reward if the reward update time elapse is small

## Summary

Division before multiplication result in loss of token reward

## Vulnerability Detail

When calcuting the reward, we are calling

```solidity
 function currentRewardsPerToken() public view returns (uint256) {
        // Rewards do not accrue if the total balance is zero
        if (totalBalance == 0) return rewardsPerTokenStored;

        // @audit
        // loss of precision
        // The number of rewards to apply is based on the reward rate and the amount of time that has passed since the last reward update
        uint256 rewardsToApply = ((block.timestamp - lastRewardUpdate) * rewardRate) /
            REWARD_PERIOD;

        // The rewards per token is the current rewards per token plus the rewards to apply divided by the total staked balance
        return rewardsPerTokenStored + (rewardsToApply * 10 ** stakedTokenDecimals) / totalBalance;
    }
```

the precision loss can be high because the accumulated reward depends on the time elapse:

(block.timestamp - lastRewardUpdate)

and the REWARD_PERIOD is hardcoded to one days:

```solidity
    /// @notice Amount of time (in seconds) that the reward rate is distributed over
    uint48 public constant REWARD_PERIOD = uint48(1 days);
```

if the time elapse is short and the currentRewardsPerToken is updated frequently, the precision loss can be heavy and even rounded to zero

the lower the token precision, the heavier the precision loss

> https://github.com/d-xo/weird-erc20#low-decimals

> Some tokens have low decimals (e.g. USDC has 6). Even more extreme, some tokens like Gemini USD only have 2 decimals.

consider as extreme case, if the reward token is Gemini USD, the reward rate is set to 1000 * 10 = 10 ** 4 = 10000

if the update reward keep getting called within 8 seconds

8 * 10000 / 86400 is already rounded down to zero and no reward is accuring for user

## Impact

Division before multiplication result in loss of token reward if the reward update time elapse is small

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L544

## Tool used

Manual Review

## Recommendation

Avoid division before multiplcation and only perform division at last