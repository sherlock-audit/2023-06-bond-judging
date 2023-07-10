ctf_sec

medium

# Strike price can be too high and cause overflow when exercise their token, then user will never exercise their option and lose their option token

## Summary

Strike price can be too high and cause overflow, then user will never exercise their option and lose their option token

## Vulnerability Detail

In the current implementation

There are three ways to set up the strike price:

The first way is [setting strike price](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L721) via ManualOTLM.sol

the second way is using the bond oracle and the strike price will be [fetched from the oracle](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L797)

The third way is user can permissionless deploy and create a option token via the Teller contract by call deploy and [set the strike price direclty](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L169)

In a call option, the user can [transfer the token in](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L277) and [mint the option token](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L296)

when creating the option token for call option, the line of code below is not called

```solidity
uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals());
```

but when exercise the option, the line of code above is always called [here](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L339)

it is possible the strike price becomes too high and amount * strikePrice exceeds the type(uint256).max and cause overflow, which revert transaction

In this case, the user can only partially exerise the option token or never able to exercise the option token

user can do nothing but wait for option to expiry and the receiver can collect their fund

## Impact

Strike price can be too high and cause overflow when exercise their token, then user will never exercise their option and lose their option token

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L339

## Tool used

Manual Review

## Recommendation

If the strike price is too high and cause overflow, consider refund the payout token token for call option
