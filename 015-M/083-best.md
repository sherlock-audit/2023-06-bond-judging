ctf_sec

medium

# Use A's staked token balance can be used to mint option token as reward for User B if the payout token equals to the stake token

## Summary

User's staked token balance can be used to mint option token as reward if the payout token equals to the stake token, can cause user to loss fund

## Vulnerability Detail

In OTLM, user can stake stakeToken in exchange for the option token minted from the payment token

when staking, we [transfer the stakedToken](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L334) in the OTLM token

```solidity
// Increase the user's stake balance and the total balance
stakeBalance[msg.sender] = userBalance + amount_;
totalBalance += amount_;

// Transfer the staked tokens from the user to this contract
stakedToken.safeTransferFrom(msg.sender, address(this), amount_);
```

before the stake or unstake or when we are calling claimReward

we are calling _claimRewards -> _claimEpochRewards -> we use payout token to mint and [create option token as reward](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L508)

```solidity
	payoutToken.approve(address(optionTeller), rewards);
	optionTeller.create(optionToken, rewards);

	// Transfer rewards to sender
	ERC20(address(optionToken)).safeTransfer(msg.sender, rewards);
```

the problem is, if the stake token and the payout token are the same token, the protocol does not distingush the balance of the stake token and the balance of payout token

**suppose both stake token and payout token are USDC**

suppose user A stake 100 USDC

suppose user B stake 100 USDC

time passed, user B accure 10 token unit reward

now user B can claimRewards, 

the protocol user 10 USDC to mint option token for B

the OTLM has 190 USDC 

if user A and user B both call emergencyUnstakeAll, whoeve call this function later will suffer a revert and he is not able to even give up the reward and claim their staked balance back

because a part of the his staked token balance is treated as the payout token to mint option token reward for other user

## Impact

If there are insufficient payout token in the OTLM, the expected behavior is that the transaction revert when claim the reward and when the code use payout token to mint option token

and in the worst case, user can call emergencyUnstakeAll to get their original staked balane back and give up their reward

however, if the staked token is the same as the payout token,

a part of the user staked token can be mistakenly and constantly mint as option token reward for his own or for other user and eventually when user call emergencyUnstakeAll, there will be insufficient token balance and transaction revert

so user will not able to get their staked token back

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L334

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L508

## Tool used

Manual Review

## Recommendation

Seperate the accounting of the staked user and the payout token or check that staked token is not payout token when creating the OTLM.sol