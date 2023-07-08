berndartmueller

medium

# Determining the next strike price with `OracleStrikeOTLM.nextStrikePrice` does not consider the Oracle's decimals, potentially resulting in an incorrect price

## Summary

Determining the next strike price with the `nextStrikePrice` function in the `OracleStrikeOTLM` contract does not consider the Oracle's decimals (`IBondOracle.decimals()`), potentially resulting in an incorrect price as the strike price is expected to be denominated in the quote token's decimals.

## Vulnerability Detail

The Oracle Strike OTLM contract (`OracleStrikeOTLM`) uses an oracle to determine the fixed strike price when a new option token is created on epoch transition. To determine the next strike price, the `nextStrikePrice` function is called. This function fetches the current price from the oracle by calling the `currentPrice` function.

As observed in the `IBondOracle` interface for the oracle, the [oracle provides a `decimals` function](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/interfaces/IBondOracle.sol#L11) returning the number of configured decimals for the price of the given quote and payout token.

However, this `decimals` function is not called, and the price returned by the oracle is not normalized to the quote token's decimals. If the oracle operates with 18 decimals for a given pair, and the used `quoteToken` is USDC with 6 decimals, the strike price will be incorrect and inflated by $10^{12}$.

## Impact

The next strike price is potentially denominated in different decimals than the quote token, which an option token is priced at, resulting in an incorrect strike price.

## Code Snippet

[src/fixed-strike/liquidity-mining/OTLM.sol#L798-L799](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L798-L799)

```solidity
797: function nextStrikePrice() public view override returns (uint256) {
798:     uint256 discountedPrice = (oracle.currentPrice(quoteToken, payoutToken) *
799:         (ONE_HUNDRED_PERCENT - oracleDiscount)) / ONE_HUNDRED_PERCENT;
800:     return discountedPrice > minStrikePrice ? discountedPrice : minStrikePrice;
801: }
```

## Tool used

Manual Review

## Recommendation

Consider normalizing the Oracle price to the quote's decimals: `discountedPrice * quoteToken.decimals()  / oracle.decimals()`
