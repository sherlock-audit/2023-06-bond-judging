ctf_sec

medium

# Too few or too much option reward token is minted if the payout token decimal miss match the staked token decimal

## Summary

Too few or too much option reward token is minted if the payout token decimal miss match the staked token decimal

## Vulnerability Detail

When creating an OTLM

the [staked token decimal is set](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L163)

```solidity
stakedTokenDecimals = stakedToken_.decimals();
```

Before minting the option token reward, the code calculated the reward:

```solidity

        // Get the rewards per token at the start of the epoch and the rewards per token at the end of the epoch (start of the next one)
        // If the epoch is the current epoch, the rewards per token at the end of the epoch is the current rewards per token stored
        uint256 rewardsPerTokenStart = epochRewardsPerTokenStart[epoch_];
        uint256 rewardsPerTokenEnd = epoch_ == epoch
            ? rewardsPerTokenStored
            : epochRewardsPerTokenStart[epoch_ + 1];

        // If the user hasn't claimed the rewards up to the start of this epoch, then they have a previous unclaimed epoch
        // External functions protect against this by their ordering, but this makes it explicit
        if (userRewardsClaimed < rewardsPerTokenStart) revert OTLM_PreviousUnclaimedEpoch();

        // If user rewards claimed is greater than or equal to the rewards per token at the end of the epoch, then the user has already claimed all rewards for the epoch
        if (userRewardsClaimed >= rewardsPerTokenEnd) return 0;

        // If not, then the user has not claimed all rewards for the epoch

        // Set the rewards per token claimed by the user to the rewards per token at the end of the epoch
        rewardsPerTokenClaimed[msg.sender] = rewardsPerTokenEnd;
        lastEpochClaimed[msg.sender] = epoch_;

        // Get the option token for the epoch
        FixedStrikeOptionToken optionToken = epochOptionTokens[epoch_];
        // If the option token has expired, then the rewards are zero
        if (uint256(optionToken.expiry()) < block.timestamp) return 0;

        // If the option token is still valid, we need to issue rewards
        uint256 rewards = ((rewardsPerTokenEnd - userRewardsClaimed) * stakeBalance[msg.sender]) /
            10 ** stakedTokenDecimals;
```

note that rewardsPerTokenStored comes from 

```solidity
  /// @notice Returns the current rewards per token value updated to the second
    function currentRewardsPerToken() public view returns (uint256) {
        // Rewards do not accrue if the total balance is zero
        if (totalBalance == 0) return rewardsPerTokenStored;

        // The number of rewards to apply is based on the reward rate and the amount of time that has passed since the last reward update
        uint256 rewardsToApply = ((block.timestamp - lastRewardUpdate) * rewardRate) /
            REWARD_PERIOD;

        // The rewards per token is the current rewards per token plus the rewards to apply divided by the total staked balance
        return rewardsPerTokenStored + (rewardsToApply * 10 ** stakedTokenDecimals) / totalBalance;
    }
```

and the core formula is

```solidity
  uint256 rewards = ((rewardsPerTokenEnd - userRewardsClaimed) * stakeBalance[msg.sender]) /
            10 ** stakedTokenDecimals;
```

basically rewardsPerTokenEnd and stakeBalance[msg.sender] are all in staked token decimals

but eventually the amount of [payout token is minted](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L504)

```solidity
// Mint the option token on the teller
// This transfers the reward amount of payout tokens to the option teller in exchange for the amount of option tokens
payoutToken.approve(address(optionTeller), rewards);
optionTeller.create(optionToken, rewards);
```

if the payoutToken decimal does not match the stake token decimal, Too few or too much option reward token is minted if the payout token decimal miss match the staked token decimal

cosndier the rewards calcuated is 10 ** 10, if the payout token is USDC, 10 ** 10 is 10000 USDC (too much option reward is minted)

but if the the payout token is ETH, the 10 ** 10 is just a tiny fraction of ETH (too few option reward token is minted)

this is espeically a issue if the staked token decimal is high but the payout token decimal is low, the contract will run out of payout token very fast

## Impact

Too few or too much option reward token is minted if the payout token decimal miss match the staked token decimal

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L504

## Tool used

Manual Review

## Recommendation

We recommend the protocol convert the rewards amount in staked token balance to payout token amount