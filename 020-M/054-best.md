berndartmueller

medium

# Minimum strike price configured in the `OracleStrikeOTLM` contract can cause an invalid strike price, preventing starting a new epoch

## Summary

The minimum strike price (`minStrikePrice`) used in the `OracleStrikeOTLM.nextStrikePrice` function can possibly be set to a value that does not prevent an invalid strike price, which will prevent deploying a new option token and thus prevent starting a new epoch.

## Vulnerability Detail

The `OracleStrikeOTLM` contract uses an external oracle (`oracle`) to determine the next strike price in the `nextStrikePrice` function. To ensure that the strike price is not too low, a minimum strike price (`minStrikePrice`) is used as a lower bound for the strike price.

`minStrikePrice` is set in the `_initialize` function and can be updated by the owner using the `setMinStrikePrice` function. However, in both instances, the `minStrikePrice` is not validated to be greater than zero or to be within the [bounds defined in lines 143-144 of the `FixedStrikeOptionTeller.deploy` function](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L143-L144). Such an invalid strike price will revert with the `Teller_InvalidParams` error, resulting in the inability to deploy a new option token and thus starting a new epoch until the owner updates the `minStrikePrice` accordingly.

## Impact

Attempting to deploy a new option token with `FixedStrikeOptionTeller.deploy` can potentially revert if the provided strike price (i.e., the result from calling `nextStrikePrice`) is considered [invalid in lines 143-144](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L143-L144).

## Code Snippet

[src/fixed-strike/liquidity-mining/OTLM.sol#L800](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L800)

The next strike price determined by the `OracleStrikeOTLM.nextStrikePrice` function uses the minimum strike price (`minStrikePrice`) as the lower bound for the strike price. If this minimum strike price is not properly set, it can lead to an invalid strike price which will prevent deploying a new option token and thus prevent starting a new epoch.

```solidity
797: function nextStrikePrice() public view override returns (uint256) {
798:     uint256 discountedPrice = (oracle.currentPrice(quoteToken, payoutToken) *
799:         (ONE_HUNDRED_PERCENT - oracleDiscount)) / ONE_HUNDRED_PERCENT;
800:     return discountedPrice > minStrikePrice ? discountedPrice : minStrikePrice;
801: }
```

[src/fixed-strike/liquidity-mining/OTLM.sol#L791](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L791)

```solidity
772: function _initialize(bytes calldata params_) internal override {
773:     (IBondOracle oracle_, uint48 oracleDiscount_, uint256 minStrikePrice_) = abi.decode(
774:         params_,
775:         (IBondOracle, uint48, uint256)
776:     );
777:
778:     // Validate parameters
779:     // Oracle discount must be less than 100% (price cannot be zero)
780:     if (oracleDiscount_ >= ONE_HUNDRED_PERCENT) revert OTLM_InvalidParams();
781:     // Oracle must be a valid contract
782:     if (address(oracle_) == address(0) || address(oracle_).code.length == 0)
783:         revert OTLM_InvalidParams();
784:
785:     // Oracle price must return a non-zero value for the quote and payout tokens
786:     if (oracle_.currentPrice(quoteToken, payoutToken) == 0) revert OTLM_InvalidParams();
787:
788:     // Set parameters
789:     oracle = oracle_;
790:     oracleDiscount = oracleDiscount_;
791: @>  minStrikePrice = minStrikePrice_; // @audit-info Missing validation
792: }
```

[src/fixed-strike/liquidity-mining/OTLM.sol#L832](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/liquidity-mining/OTLM.sol#L832)

```solidity
831: function setMinStrikePrice(uint256 minStrikePrice_) external onlyOwner requireInitialized {
832:     minStrikePrice = minStrikePrice_; // @audit-info Missing validation
833: }
```

## Tool used

Manual Review

## Recommendation

Consider validating the minimum strike price to be greater than zero and to be [within the bounds defined in `FixedStrikeOptionTeller.deploy`](https://github.com/sherlock-audit/2023-06-bond/blob/main/options/src/fixed-strike/FixedStrikeOptionTeller.sol#L143-L144) to prevent using a `nextStrikePrice` which leads to an invalid strike price that reverts in the `FixedStrikeOptionTeller.deploy` function.
