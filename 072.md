berndartmueller

high

# Funds can be stolen from the `FixedStrikeOptionTeller` contract by creating put option tokens without providing collateral

## Summary

Due to a rounding error when calculating the `quoteAmount` in the `create` function of the `FixedStrikeOptionTeller` contract, it is possible to create (issue) option tokens without providing the necessary collateral. A malicious receiver can exploit this to steal funds from the `FixedStrikeOptionTeller` contract.

## Vulnerability Detail

Anyone can create (issue) put option tokens with the `create` function in the `FixedStrikeOptionTeller` contract. However, by specifying a very small `amount_`, the `quoteAmount` calculation in line 283 can potentially round down to zero. This is the case if the result of the multiplication in line 283, $amount * strikePrice$ is smaller than $10^{decimals}$, where `decimals` is the number of decimals of the payout token.

For example, assume the following scenario:

| Parameter                | Description                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------ |
| Quote token              | $USDC. 6 decimals                                                                                |
| Payout token             | $GMX. 18 decimals                                                                                |
| $payoutToken_{decimals}$ | 18 decimals                                                                                      |
| $amount$                 | `1e10`. Amount (`amount_`) supplied to the `create` function, in payout token decimal precision. |
| $strikePrice$            | `50e6` ~ 50 USD. The strike price of the option token, in quote tokens.                              |

_[Please note that the option token has the same amount of decimals as the payout token.](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L221)_

Calculating `quoteAmount` leads to the following result:

$$
\begin{align}
quoteAmount &= \dfrac{amount * strikePrice}{10^{payoutToken_{decimals}}} \\
&= \dfrac{1e10 * 50e6}{10^{18}} \\
&= \dfrac{5e17}{10^{18}} \\
&= \dfrac{5 * 10^{17}}{10^{18}} \\
&= 0
\end{align}
$$

As observed, the result is rounded down to zero due to the numerator being smaller than the denominator.

This results in **0** quote tokens to be transferred from the caller to the contract, and in return, the caller receives `1e10` ($amount$) option tokens. This process can be repeated to mint an arbitrary amount of option tokens for free.

## Impact

Put options can be minted for free without providing the required `quoteToken` collateral.

This is intensified by the fact that a malicious receiver, which anyone can be, can exploit this issue by deploying a new option token (optionally with a very short expiry), repeatedly minting free put options to accumulate option tokens, and then, once the option expires, call `reclaim` to receive quote token collateral.

This collateral, however, was supplied by other users who issued (created) option tokens with the same quote token. Thus, the malicious receiver can drain funds from other users and cause undercollateralization of those affected option tokens.

## Code Snippet

[src/fixed-strike/FixedStrikeOptionTeller.sol#L283](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L283)

```solidity
236: function create(
237:     FixedStrikeOptionToken optionToken_,
238:     uint256 amount_
239: ) external override nonReentrant {
...      // [...]
268:
269:     // Transfer in collateral
270:     // If call option, transfer in payout tokens equivalent to the amount of option tokens being issued
271:     // If put option, transfer in quote tokens equivalent to the amount of option tokens being issued * strike price
272:     if (call) {
...          // [...]
281:     } else {
282:         // Calculate amount of quote tokens required to mint
283: @>      uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals()); // @audit-issue Rounds down
284:
285:         // Transfer quote tokens from user
286:         // Check that amount received is not less than amount expected
287:         // Handles edge cases like fee-on-transfer tokens (which are not supported)
288:         uint256 startBalance = quoteToken.balanceOf(address(this));
289:         quoteToken.safeTransferFrom(msg.sender, address(this), quoteAmount);
290:         uint256 endBalance = quoteToken.balanceOf(address(this));
291:         if (endBalance < startBalance + quoteAmount)
292:             revert Teller_UnsupportedToken(address(quoteToken));
293:     }
294:
295:     // Mint new option tokens to sender
296:     optionToken.mint(msg.sender, amount_);
297: }
```

## Tool used

Manual Review

## Recommendation

Consider adding a check after line 283 to ensure `quoteAmount` is not zero.
