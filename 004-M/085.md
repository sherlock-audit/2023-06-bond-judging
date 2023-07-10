ctf_sec

medium

# User cannot emergencyUnstake in certain case because the staked token balance is treated as the payout balance if the payout token equals to the staked token

## Summary

User cannot emergencyUnstake in certain case because the staked token balance is treated as the paytoken balance 

## Vulnerability Detail

In OTLM.sol, there is a function for user to emergencyUnstake

```solidity
    /// @notice Withdraw entire balance of staking tokens without updating or claiming outstanding rewards.
    /// @notice Rewards will be lost if stake is withdrawn using this function. Only for emergency use.
    function emergencyUnstakeAll() external nonReentrant {
```

in the worst case, user should always get his original staked back

However, if the staked token and the pay token are the same token

even a [good fatih owner of the OTLM](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L578) can also withdrawal all token from the contract because the contract does not keep track of the staking balance and the payout token balance seperately

the comment indicate the owner should withdrawal the payout token, not user's staked token

```solidity
function withdrawPayoutTokens(address to_, uint256 amount_) external onlyOwner {
	// Revert if the amount is greater than the balance
	// Transfer will check this, but we provide a more helpful error message
	if (amount_ > payoutToken.balanceOf(address(this)) || amount_ == 0)
		revert OTLM_InvalidAmount();

	// Withdraws payout tokens from the contract
	payoutToken.safeTransfer(to_, amount_);
}
```

and user's emergencyUnstakeAll would revert in insufficient balance

## Impact

User cannot emergencyUnstake in certain case because the staked token balance is treated as the paytoken balance 

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L578

## Tool used

Manual Review

## Recommendation

Keep track of the staked balance and the payout token balance seperately or validate the staked token is not the payout token