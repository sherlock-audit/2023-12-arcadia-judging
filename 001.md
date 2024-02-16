Powerful Slate Hawk

high

# Account in some cases will become liquidatable after an upgrade

## Summary
Two different registries can have different risk parameters `minUsdValue`. That means that some assets that were higher than `minUsdValue1` but lower than `minUsdValue2` will affect the total collateral value of the account. So, an account's `getCollateralValue` will decrease, thus an account can become liquidatable after an upgrade.
## Vulnerability Detail
```solidity
    function getValuesInUsd(
        address creditor,
        address[] calldata assets,
        uint256[] calldata assetIds,
        uint256[] calldata assetAmounts
    ) public view sequencerNotDown(creditor) returns (AssetValueAndRiskFactors[] memory valuesAndRiskFactors) {
        uint256 length = assets.length;
        valuesAndRiskFactors = new AssetValueAndRiskFactors[](length);

        uint256 minUsdValue = riskParams[creditor].minUsdValue;
        for (uint256 i; i < length; ++i) {
            (
                valuesAndRiskFactors[i].assetValue,
                valuesAndRiskFactors[i].collateralFactor,
                valuesAndRiskFactors[i].liquidationFactor
            ) = IAssetModule(assetToAssetModule[assets[i]]).getValue(creditor, assets[i], assetIds[i], assetAmounts[i]);
            // If asset value is too low, set to zero.
            // This is done to prevent dust attacks which may make liquidations unprofitable.
            if (valuesAndRiskFactors[i].assetValue < minUsdValue) valuesAndRiskFactors[i].assetValue = 0;
        }
    }

```
[src/Registry.sol#L663](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/Registry.sol#L663)
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Add healthy check
```diff

    function upgradeAccount(address newImplementation, address newRegistry, uint256 newVersion, bytes calldata data)
        external
        onlyFactory
        nonReentrant
        notDuringAuction
        updateActionTimestamp
    {
        // Cache old parameters.
        address oldImplementation = _getAddressSlot(IMPLEMENTATION_SLOT).value;
        address oldRegistry = registry;
        uint256 oldVersion = ACCOUNT_VERSION;

        // Store new parameters.
        _getAddressSlot(IMPLEMENTATION_SLOT).value = newImplementation;
        registry = newRegistry;

        // Prevent that Account is upgraded to a new version where the Numeraire can't be priced.
        if (newRegistry != oldRegistry && !IRegistry(newRegistry).inRegistry(numeraire)) {
            revert AccountErrors.InvalidRegistry();
        }

        // If a Creditor is set, new version should be compatible.
        if (creditor != address(0)) {
            (bool success,,,) = ICreditor(creditor).openMarginAccount(newVersion);
            if (!success) revert AccountErrors.InvalidAccountVersion();
        }

        // Hook on the new logic to finalize upgrade.
        // Used to eg. Remove exposure from old Registry and add exposure to the new Registry.
        // Extra data can be added by the Factory for complex instructions.
        this.upgradeHook(oldImplementation, oldRegistry, oldVersion, data);

        // Event emitted by Factory.
+        if (isAccountUnhealthy()) revert AccountErrors.AccountUnhealthy();
    }
```