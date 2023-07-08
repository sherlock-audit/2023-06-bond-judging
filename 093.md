ctf_sec

high

# OTLM: Stakers unable to claim their rewards

## Summary
If epoch count grows too large then stakers will not be able to claim their rewards using `claimRewards` function. Also the `claimNextEpochRewards` function will be of no use.

## Vulnerability Detail
The claiming of rewards mechanism has these nuances:
1. The `claimRewards` function tries to call `_claimEpochRewards` for every epoch from `0` to current `epoch`.
2. Since there is a possibility of `claimRewards` reverting due to block gas limit, a user has the option to call `claimNextEpochRewards` to claim rewards for next unclaimed epoch.
3. Calling `_claimEpochRewards` for any epoch number less than the epoch number at which user staked is a no-op, i.e., no rewards are distributed and no states are updated.

A valid scenario of `#3` is, if a user staked at epoch `#10` then calling `_claimEpochRewards` for epoch `0-9` is a no-op.

A combination of above discussed mechanisms leads to a `very likely` scenario in which user becomes unable to claim his rewards forever.

Bug Scenario:
- Bob stakes his tokens in OTLM contract at epoch 10.
- Time passes by and the number of epochs passed becomes too high.
- Now due to high epoch count, Bob cannot call `claimRewards` function as it reverts with out of gas error due to the unbounded `for` loop.
- Bob tries to call the `claimNextEpochRewards` which internally calls `_claimEpochRewards` for epoch `0`. Due to [this](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L485) statement the `_claimEpochRewards` returns without updating any contract state.
- Hence in `claimNextEpochRewards` function no states are updated and no rewards are sent. Further calling the `claimNextEpochRewards` function will result in similar `no-op`.

Thus now user cannot call `claimRewards`, and `claimNextEpochRewards` is a no-op. This leaves no way for Bob to claim his legimately accrued rewards. Resulting in complete loss of rewards for Bob.

### POC
This test case was added in `2023-06-bond/options/test/FixedStrikeOTLM.t.sol` and ran using `forge test --mt test_audit` command

```solidity
    function test_audit_claimRewards() public {
        _manualStrikeOTLM();

        // 10 epochs passes by before Bob stakes
        for (uint i; i < 9 ; i++) {
            vm.prank(alice);
            vm.warp(block.timestamp + 1 days);
            motlm.triggerNextEpoch();
        }

        // Bob stakes at epoch 10
        uint256 bobBalance = abc.balanceOf(bob);
        vm.prank(bob);
        motlm.stake(bobBalance, ZERO_BYTES);

        // 10 epochs passes by after Bob stakes
        for (uint i; i < 100 ; i++) {
            vm.prank(alice);
            vm.warp(block.timestamp + 1 days);
            motlm.triggerNextEpoch();
        }

        // Now since epoch count has grown, claimRewards will revert due to block gas limit.
        vm.expectRevert();
        vm.prank(bob);
        motlm.claimRewards{gas: 100000}();

        // So the only option for Bob is to call claimNextEpochRewards.

        // Bob tries to claim next epoch reward
        vm.prank(bob);
        motlm.claimNextEpochRewards();

        // No rewards are distributed to Bob
        for (uint48 i = 1; i < motlm.epoch(); i++) {
            assertEq(motlm.epochOptionTokens(i).balanceOf(bob), 0);
        }
    }
```

## Impact
In case the epoch count grows (which will eventually always happen) any staker will not be able to claim his rewards.

The impact is high (loss of rewards) & likelyhood is also high (epoch count will always grow).


## Code Snippet
https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L485

## Tool used

Manual Review

## Recommendation
When a user stakes for the first time, consider updating the `lastEpochClaimed` to current epoch so that the `claimNextEpochRewards` calls `_claimEpochRewards` with `lastEpochClaimed` epoch rather than `0` epoch.

Something like this:
```solidity
    function stake(    ...    ) external nonReentrant requireInitialized updateRewards tryNewEpoch {
        ...
        if (userBalance > 0) {
            ...
        } else {
            // Initialize the rewards per token claimed for the user to the stored rewards per token
            rewardsPerTokenClaimed[msg.sender] = rewardsPerTokenStored;
            lastEpochClaimed[msg.sender] = epoch;                                      // --------> mitigation
        }

        ...
    }
```