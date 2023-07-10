lil.eth

high

# Creation of Options At Expiry Time Can Lead to Loss of Funds

## Summary

A vulnerability in the option creation mechanism allows for the creation of options at the exact expiry time (`block.timestamp == expiry`). This leaves the option holder without any time to exercise the option, causing potential loss of the holder's collateral.
Even worse, if just after `create()` tx , receiver call `reclaim()` , payout or quote token amount just deposited to create the option token will be sent to receiver

## Vulnerability Detail

The function `create` allows users to mint a new option. It verifies the `expiry` time, but it only checks whether the expiry is in the past (`if (uint256(expiry) < block.timestamp) revert Teller_OptionExpired(expiry);`). This allows the creation of options when `block.timestamp == expiry` : 
```solidity
// Revert if expiry is in the past
if (uint256(expiry) < block.timestamp) revert Teller_OptionExpired(exppiry);
```

However, in the exercise function, an option holder is prohibited from exercising the option at or after the expiry time (if (`uint48(block.timestamp) >= expiry) revert Teller_OptionExpired(expiry);`) : 
```solidity
// Validate that option token is not expired
if (uint48(block.timestamp) >= expiry) revert Teller_OptionExpired(expiry);
```
This leaves a window where an option can be created but never exercised. If an option is created at `block.timestamp == expiry`, the holder won't have time to exercise the option.

And in the meantime, receiver can call `reclaim()` function at `block.timestamp == expiry` : 
```solidity
if (uint48(block.timestamp) < expiry) revert Teller_NotExpired(expiry);
```

## Impact

The option holder could potentially lose their collateral. As soon as the option is created at expiry == block.timestamp, the receiver is allowed to reclaim the funds, causing the holder to lose their entire collateral without being able to exercise the option. This could lead to significant loss if done unintentionally or manipulated by a malicious actor

## Code Snippet

- create() : https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L236
- reclaim() : https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395

## Tool used

Manual Review

## Recommendation

Modify the create function to refuse the creation of options at the expiry time. This can be done by changing the condition that checks the expiry time to `if (uint256(expiry) <= block.timestamp) revert Teller_OptionExpired(expiry);`. This change will ensure that there is always a time window for the option holder to exercise the option after its creation, protecting them from potential loss