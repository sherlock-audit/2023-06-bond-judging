ctf_sec

medium

# FixedStrikeOptionTeller: create can be invoked when block.timestamp == expiry but exercise reverts

## Summary
In `FixedStrikeOptionTeller` contract, new option tokens can be minted when `block.timestamp == expiry` but these option tokens cannot be exercised even in the same transaction.

## Vulnerability Detail
The `create` function has this statement:
```solidity
        if (uint256(expiry) < block.timestamp) revert Teller_OptionExpired(expiry);
```

The `exercise` function has this statement:
```solidity
        if (uint48(block.timestamp) >= expiry) revert Teller_OptionExpired(expiry);
```
Notice the `>=` operator which means when `block.timestamp == expiry` the `exercise` function reverts.

The `FixedStrikeOptionTeller.create` function is invoked whenever a user claims his staking rewards using `OTLM.claimRewards` or `OTLM.claimNextEpochRewards`. ([here](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L505))

So if a user claims his rewards when `block.timestamp == expiry` he receives the freshly minted option tokens but he cannot exercise these option tokens even in the same transaction (or same block).

Moreover, since the receiver do not possess these freshly minted option tokens, he cannot `reclaim` them either (assuming `reclaim` function contains the currently missing `optionToken.burn` statement).


## Impact
Option token will be minted to user but he cannot exercise them. Receiver cannot reclaim them as he doesn't hold that token amount.

This leads to loss of funds as the minted option tokens become useless. Also the scenario of users claiming at expiry is not rare.

## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L267
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L336

## Tool used

Manual Review

## Recommendation
Consider maintaining a consistent timestamp behaviour. Either prevent creation of option tokens at expiry or allow them to be exercised at expiry.