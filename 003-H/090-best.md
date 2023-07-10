ctf_sec

high

# All funds can be stolen from FixedStrikeOptionTeller using a token with malicious decimals

## Summary

`FixedStrikeOptionTeller` is a single contract which deploys multiple option tokens. Hence this single contract holds significant payout/quote tokens as collateral. Also the `deploy`, `create` & `exercise` functions of this contract can be called by anyone.

This mechanism can be exploited to drain `FixedStrikeOptionTeller` of all tokens.


## Vulnerability Detail
This is how the create functions looks like:
```solidity
    function create(
        FixedStrikeOptionToken optionToken_,
        uint256 amount_
    ) external override nonReentrant {
        ...
        if (call) {
            ...
        } else {
            uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals());
            ...
            quoteToken.safeTransferFrom(msg.sender, address(this), quoteAmount);
            ...
        }

        optionToken.mint(msg.sender, amount_);
    }
```

exercise function:
```solidity
    function exercise(
        FixedStrikeOptionToken optionToken_,
        uint256 amount_
    ) external override nonReentrant {
        ...
        uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals());

        if (msg.sender != receiver) {
            ...
        }

        optionToken.burn(msg.sender, amount_);

        if (call) {
            ...
        } else {
            quoteToken.safeTransfer(msg.sender, quoteAmount);
        }
    }
```

Consider this attack scenario:

Let's suppose the `FixedStrikeOptionTeller` holds some DAI tokens.

- An attacker can create a malicious payout token of which he can control the `decimals`.

- The attacker calls `deploy` to create an option token with malicious payout token and DAI as quote token and `put` option type

- Make `payoutToken.decimals` return a large number and call `FixedStrikeOptionTeller.create` with input X. [Here `quoteAmount` will be calculated as `0`](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L283). 

```solidity
// Calculate amount of quote tokens required to mint
uint256 quoteAmount = amount_.mulDiv(strikePrice, 10 ** payoutToken.decimals());

// Transfer quote tokens from user
// Check that amount received is not less than amount expected
// Handles edge cases like fee-on-transfer tokens (which are not supported)
uint256 startBalance = quoteToken.balanceOf(address(this));
quoteToken.safeTransferFrom(msg.sender, address(this), quoteAmount);
```

So 0 DAI will be pulled from the attacker's account but he will receive X option token.

- Make `payoutToken.decimals` return a small value and call `FixedStrikeOptionTeller.exercise` with X input. Here `quoteAmount` will be calculated as a very high number (which represents number of DAI tokens). So he will receive huge amount of DAI against his X option tokens when [exercise the option](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L339) or when [reclaim the token](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L434)

```solidity
// Transfer remaining collateral to receiver
uint256 amount = optionToken.totalSupply();
if (call) {
	payoutToken.safeTransfer(receiver, amount);
} else {
	// Calculate amount of quote tokens equivalent to amount at strike price
	uint256 quoteAmount = amount.mulDiv(strikePrice, 10 ** payoutToken.decimals());
	quoteToken.safeTransfer(receiver, quoteAmount);
}
```

Hence, the attacker was able to drain all DAI tokens from the `FixedStrikeOptionTeller` contract. The same mechanism can be repeated to drain all other ERC20 tokens from the `FixedStrikeOptionTeller` contract by changing the return value of the decimal external call

## Impact

Anyone can drain `FixedStrikeOptionTeller` contract of all ERC20 tokens. The cost of attack is negligible (only gas cost). 

High impact, high likelyhood.

## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L283

https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L339

## Tool used

Manual Review

## Recommendation
Consider storing the `payoutToken.decimals` value locally instead of fetching it real-time on all `exercise` or `reclaim` calls.

or support payout token and quote token whitelist, if the payout token and quote token are permissionless created, there will always be high risk
