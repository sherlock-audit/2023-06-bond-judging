ctf_sec

medium

# Loss of option token from Teller and reward from OTLM if L2 sequencer goes down

## Summary

Loss of option token from Teller and reward from OTLM if L2 sequencer goes down

## Vulnerability Detail

In the current implementation, if the option token expires, the user is not able to [exerise the option at strike price](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L336)

```solidity
    // Validate that option token is not expired
        if (uint48(block.timestamp) >= expiry) revert Teller_OptionExpired(expiry);
```

if the option token expires, the user lose rewards from OTLM as well when [claim the reward](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L496)

```solidity
    function _claimRewards() internal returns (uint256) {
        // Claims all outstanding rewards for the user across epochs
        // If there are unclaimed rewards from epochs where the option token has expired, the rewards are lost

        // Get the last epoch claimed by the user
        uint48 userLastEpoch = lastEpochClaimed[msg.sender];

```

and

```solidity
    // If the option token has expired, then the rewards are zero
        if (uint256(optionToken.expiry()) < block.timestamp) return 0;
```

And in the onchain context, the protocol intends to deploy the contract in arbitrum and optimsim

```solidity
Q: On what chains are the smart contracts going to be deployed?
Mainnet, Arbitrum, Optimism
```

However, If Arbitrum and optimism layer 2 network, the sequencer is in charge of process the transaction

For example, the recent optimism bedrock upgrade cause the sequencer not able to process transaction for a hew hours

https://cryptopotato.com/optimism-bedrock-upgrade-release-date-revealed/

> Bedrock Upgrade
According to the official announcement, the upgrade will require 2-4 hours of downtime for OP Mainnet, during which there will be downtime at the chain and infrastructure level while the old sequencer is spun down and the Bedrock sequencer starts up.

> Transactions, deposits, and withdrawals will also remain unavailable for the duration, and the chain will not be progressing. While the read access across most OP Mainnet nodes will stay online, users may encounter a slight decrease in performance during the migration process.

In Arbitrum

https://thedefiant.io/arbitrum-outage-2

> Arbitrum Goes Down Citing Sequencer Problems
Layer 2 Arbitrum suffers 10 hour outage.

and

https://beincrypto.com/arbitrum-sequencer-bug-causes-temporary-transaction-pause/

> Ethereum layer-2 (L2) scaling solution Arbitrum stopped processing transactions on June 7 because its sequencer faced a bug in the batch poster. The incident only lasted for an hour.

If the option expires during the sequencer down time, the user basically have worthless option token because they cannot exercise the option at strike price

the user would lose his reward as option token from OTLM.sol, which defeats the purpose of use OTLM to incentive user to provide liquidity 

## Impact

Loss of option token from Teller and reward from OTLM if L2 sequencer goes down

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L336

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L496

## Tool used

Manual Review

## Recommendation

chainlink has a sequencer up feed

https://docs.chain.link/data-feeds/l2-sequencer-feeds

consider integrate the up time feed and give user extra time to exercise token and claim option token reward if the sequencer goes down