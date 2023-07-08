BenRai

high

# `optionTokens` can be expired even though the epoch is not over

## Summary

When deploying an `optionToken` the parameter `expiry` is rounded down to the “nearest day at 0000 UTC” but since the end of an epoch is calculated by the `epochDuration` and the exact time the epoch has stared and the `optionToken` was created this can lead to an epoch still being active but the corresponding `optionToken` to be already expired. 

## Vulnerability Detail

When starting a new epoch, the variable `epochStart` is set to the current time (`block.timestamp`) and the end of the epoch is calculated by adding the `epochDuration` to the `epochStart` variable. 

The `optionToken` of the new epoch is deployed with the parameter `expire` calculated based on the current time stamp, the `timeUntilEligible` and the `eligibleDuration`. (`uint48(block.timestamp) + timeUntilEligible + eligibleDuration`). The final expiration date of the optionToken is rounded down to the “nearest day at 0000 UTC” before the token is deployed.

Since the `epochDuration` can be as close as 1 second to the sum of `timeUntilEligible + eligibleDuration` this can lead to an epoch still being active but its `optionToken` to be already expired.

Example:

epochDuration = 7 days
timeUntilEligible = 0
eligibleDuration = 7 days + 12 hours


New epoch is launched on the 01.01.2024 at 11:45 am.

=>
epochStart = block.timestamp  = 01.01.2024 at 11:45 am
epochEnd = epochStart + epochDuration = 08.01.2024 at 11:45 am

`initial expire` = block.timestamp + timeUntilEligible + eligibleDuration = 08.01.2024 at 11:45 pm

`final expire` after rounding down = uint48(`initial expire`/ 1day) * 1 day = 08.01.2024 at 00:00 am

The epoch is still active between `final expire` and `epochEnd` even though the option has already expired.

## Impact

Users that wait until the epoch has ended to claim their rewards expecting the options to be exercisable for 12 hours after the epoch end cannot claim their options since they are expired already and lose out on all the value the options would have had which can be significant depending on the current price of the `payoutToken`

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L514-L534

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L122


https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L605-L611

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L629-L643


## Tool used

Manual Review

## Recommendation

The expiration of the `optionTokens` should be rounded up instead of down. This would increase the time an option can be redeemed long enough to prevent the scenario described above.
