ctf_sec

medium

# Flashloan can be used to bypass the token allow list check

## Summary

Flashloan can be used to bypass the token allow list check

## Vulnerability Detail

When staking, the code check if the tokenAllow list is set

and if the tokenAllow list is set, the code [check if msg.sender is eligible to call stake](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/fixed-strike/liquidity-mining/OTLM.sol#L313)

```solidity
    // If allowlist configured, check if user is allowed to stake
    if (address(allowlist) != address(0)) {
        if (!allowlist.isAllowed(msg.sender, proof_)) revert OTLM_NotAllowed();
    }
```

calling

```solidity
    function isAllowed(address user_, bytes calldata proof_) external view override returns (bool) {
        // External proof data isn't needed for this implementation

        // Get the allowlist token and balance threshold for the sender contract
        TokenCheck memory check = checks[msg.sender];

        // Return whether or not the user passes the balance threshold check
        return check.token.balanceOf(user_) >= uint256(check.threshold);
    }
```

the [balance check ](https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/periphery/TokenAllowlist.sol#L46)

use spot balance to check the caller of the stake function

such restriction can bypass by using flashloan to inflate and borrow the caller's balance and render the token allow list ineffective

## Impact

token allow list check can be bypassed and render the token allow list ineffective

## Code Snippet

https://github.com/sherlock-audit/2023-06-bond/blob/fce1809f83728561dc75078d41ead6d60e15d065/options/src/periphery/TokenAllowlist.sol#L46

## Tool used

Manual Review

## Recommendation

Implement merkle proof, or whitelist the caller instead of using balanceOf check