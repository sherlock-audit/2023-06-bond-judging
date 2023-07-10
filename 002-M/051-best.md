berndartmueller

medium

# Low-value quote tokens cannot be used for high strike prices

## Summary

Low-value quote tokens cannot be used to quote high-value strike prices due to the `priceDecimals` bounds check in the `deploy` function of the `FixedStrikeOptionTeller` contract reverting.

## Vulnerability Detail

Deploying a new ERC-20 option token restricts the provided strike price (`strikePrice`) in lines 142-144 of the `FixedStrikeOptionTeller.deploy` function from either being zero or, if non-zero, having too few or too many decimals (relative to the token decimal amount).

For example, consider `Shiba Inu` (`SHIB`, with 18 decimals) as the quote token, which is a very low USD-valued token (currently ~0,000005 USD) and intended to be used to quote the strike price of 50.000 USD for a given payout token (with 18 decimals). The strike price would be `10_000_000_000e18` `SHIB` (ten billion) tokens. This strike price would be rejected by the `deploy` function, as the strike price has `10 + 18 = 28` decimals, `28 - 18 = 10` decimals relative to the quote token's 18 decimals. Those 10 decimals exceed the upper bound of 9 decimals.

## Impact

Certain ERC-20 tokens with a low value compared to the payout token cannot be used as quote tokens and will revert when attempting to deploy an option token.

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

Consider relaxing the `priceDecimals` bounds in the `deploy` function, especially as the token name bytes limit of 32 bytes is not fully exhausted by increasing the possible strike price length by 1. The bytes utilization explanation in the comment in [FixedStrikeOptionTeller.sol#L643](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L643) demonstrates a maximum of 31 bytes are currently used for the token name, leaving 1 additional byte to use for a double-digit exponent.
