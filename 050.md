berndartmueller

medium

# Strike price bounds check is ineffective for quote tokens with few decimals

## Summary

The `priceDecimals` bounds check in the `deploy` function of the `FixedStrikeOptionTeller` contract cannot prevent small strike prices, e.g., `1 wei`, if the quote token has low decimals (e.g., `USDC`, `GUSD` with 2 decimals). The strike price would be allowed, leading to potential precision issues down the road.

## Vulnerability Detail

Deploying a new ERC-20 option token restricts the provided strike price (`strikePrice`) in lines 142-144 of the `FixedStrikeOptionTeller.deploy` function from either being zero or, if non-zero, having too few or too many decimals (relative to the token decimal amount).

However, the currently used fixed bounds of **-9** and **9** decimals are not effective in preventing small strike prices (e.g., `1 wei`) for quote tokens with few decimals (e.g. `USDC` with 6 decimals, `GUSD` with 2 decimals).

Consider the example with a USDC as the quote token:

- Quote token decimals: `6`
- The strike price is `1 wei`

`priceDecimals` is **-6**, within the bounds of **-9** and **9**, thus allowed to be used as the strike price.

## Impact

Very small strike prices (e.g., `1 wei`) for quote tokens with few decimals (e.g., `USDC` with 6 decimals , `GUSD` with 2 decimals) are allowed, potentially leading to precision issues down the road and causing unexpected behavior.

## Code Snippet

[src/fixed-strike/FixedStrikeOptionTeller.sol#L142-L144](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L142-L144)

```solidity
107: function deploy(
108:     ERC20 payoutToken_,
109:     ERC20 quoteToken_,
110:     uint48 eligible_,
111:     uint48 expiry_,
112:     address receiver_,
113:     bool call_,
114:     uint256 strikePrice_
115: ) external override nonReentrant returns (FixedStrikeOptionToken) {
...      // [...]
140:
141:     // Revert if strike price is zero or out of bounds
142: @>  int8 priceDecimals = _getPriceDecimals(strikePrice_, quoteToken_.decimals());
143: @>  if (strikePrice_ == 0 || priceDecimals > int8(9) || priceDecimals < int8(-9))
144: @>      revert Teller_InvalidParams(6, abi.encodePacked(strikePrice_));
145:
146:     // Create option token if one doesn't already exist
147:     // Timestamps are truncated above to give canonical version of hash
148:     bytes32 optionHash = _getOptionTokenHash(
149:         payoutToken_,
150:         quoteToken_,
151:         eligible_,
152:         expiry_,
153:         receiver_,
154:         call_,
155:         strikePrice_
156:     );
```

[src/fixed-strike/FixedStrikeOptionTeller.\_getPriceDecimals](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L678)

```solidity
678: function _getPriceDecimals(uint256 price_, uint8 tokenDecimals_) internal pure returns (int8) {
679:     int8 decimals;
680:     while (price_ >= 10) {
681:         price_ = price_ / 10;
682:         decimals++;
683:     }
684:
685:     // Subtract the stated decimals from the calculated decimals to get the relative price decimals.
686:     // Required to do it this way vs. normalizing at the beginning since price decimals can be negative.
687:     return decimals - int8(tokenDecimals_);
688: }
```

## Tool used

Manual Review

## Recommendation

Consider using relative lower- and upper bounds for the strike price decimals instead of using **-9** and **9** as fixed bounds to more effectively enforce a lower bound.

For example, if the token's decimals are **6**, the relative strike price decimals should be at least between **-3** and **3**. Additionally, the upper bound could be more relaxed and even removed.
