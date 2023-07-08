ctf_sec

high

# All fund from Teller contract can be drained because a malicious receiver can call reclaim repeatedly

## Summary

All fund from Teller contract can be drained because a malicious receiver can call reclaim repeatedly

## Vulnerability Detail

When [mint an option token](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L296), the user is required to transfer the [payout token for a call option](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L277) or [quote token for a put option](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L289)

if after the expiration, the receiver can call reclaim to claim the payout token if the option type is call or claim the quote token if the option type is put

however, the root cause is when reclaim the token, the corresponding option is not burnt ([code](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395))

```solidity
       // Revert if caller is not receiver
        if (msg.sender != receiver) revert Teller_NotAuthorized();

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

the Teller contract is likely to hold fund from multiple option token

a malicious actor can create call [Teller#deploy](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L107) and set a receiver address that can control by himself

and then wait for the option expiry and repeated call reclaim to steal the fund from the Teller contract

## Impact

All fund from Teller contract can be drained because a malicious receiver can call reclaim repeatedly

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L395

## Tool used

Manual Review

## Recommendation

Burn the corresponding option burn when reclaim the fund