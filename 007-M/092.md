ctf_sec

medium

# FixedStrikeOptionTeller: Receiver can only reclaim entire suply of option token and not a partial option token amount

## Summary
In `FixedStrikeOptionTeller` contract the `receiver` of an option token cannot reclaim partial option token amount. This can lead to collateral tokens (payout or quote tokens) getting stuck in `FixedStrikeOptionTeller` contract permanently.

## Vulnerability Detail
The `FixedStrikeOptionTeller.reclaim` function looks like this:
```solidity
    function reclaim(FixedStrikeOptionToken optionToken_) external override nonReentrant {
        ...

        uint256 amount = optionToken.totalSupply();
        if (call) {
            payoutToken.safeTransfer(receiver, amount);
        } else {
            uint256 quoteAmount = amount.mulDiv(strikePrice, 10 ** payoutToken.decimals());
            quoteToken.safeTransfer(receiver, quoteAmount);
        }
    }
```

It can be seen that the `reclaim` function always tries to reclaim the entire supply of `optionToken`.

Note: The above function also has a missing `optionToken.burn` statement which I have reported as a separate issue. For this issue, I will assume that the missing statement is present at the end of `reclaim` function.

Since the options tokens can be minted by anyone using the public `create` function, in fact, option tokens are generally minted to stakers of OTLM contract, the `receiver` will certainly not hold the total supply of option tokens. Hence the reclaiming and burning of entire option token supply can revert the transaction.

## Impact
The receiver will not be able to reclaim total supply of option tokens due to transaction getting reverted. Moreover the receiver does not have the option to reclaim a partial option token amount.

This leads to collateral tokens getting stuck in the `FixedStrikeOptionTeller` contract permanently.


## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L429-L436

## Tool used

Manual Review

## Recommendation
Consider adding a mechanism for the receiver to reclaim partial option token amounts.