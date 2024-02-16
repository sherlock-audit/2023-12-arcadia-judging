Bumpy Concrete Mouse

medium

# `riskFactor` is incorrectly double applied to `StakeStargateAM` assets

## Summary

When applying `riskFactor` in `StateStargateAM` it is applied to the entire position, including the Stargate LP. Stargate LP is already a derived asset and therefore will already have `riskFactor` applied to it. The recursive nature of this pricing causes it to apply twice to the underlying asset.

## Vulnerability Detail

When querying the poolInfo of [LPStakingTime](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#readContract) we see that the LP token returned by `poolInfo(1)` isn't USDC directly but rather USDC LP.

[AbstractStakingAM.sol#L256-L272](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L256-L272)

    if (valueInUsd > 0) {
        unchecked {
            collateralFactor = (
                valueStakedAsset * rateUnderlyingAssetsToUsd[0].collateralFactor
                    + valueRewardAsset * rateUnderlyingAssetsToUsd[1].collateralFactor
            ) / valueInUsd;
            liquidationFactor = (
                valueStakedAsset * rateUnderlyingAssetsToUsd[0].liquidationFactor
                    + valueRewardAsset * rateUnderlyingAssetsToUsd[1].liquidationFactor
            ) / valueInUsd;
        }
    }


    // Lower risk factors with the protocol wide risk factor.
    uint256 riskFactor = riskParams[creditor].riskFactor;
    collateralFactor = riskFactor.mulDivDown(collateralFactor, AssetValuationLib.ONE_4);
    liquidationFactor = riskFactor.mulDivDown(liquidationFactor, AssetValuationLib.ONE_4);

We see in `abstractStakingAM`, that the `riskFactor` is applied to the combined factors of both the LP and reward token.

[StargateAM.sol#L216-L222](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StargateAM.sol#L216-L222)

    valueInUsd = underlyingAssetsAmounts[0].mulDivDown(rateUnderlyingAssetsToUsd[0].assetValue, 1e18);

    // Lower risk factors with the protocol wide risk factor.
    uint256 riskFactor = riskParams[creditor].riskFactor;
    collateralFactor = riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[0].collateralFactor, AssetValuationLib.ONE_4);
    liquidationFactor =
        riskFactor.mulDivDown(rateUnderlyingAssetsToUsd[0].liquidationFactor, AssetValuationLib.ONE_4);

We also see that in `StargateAM` that the `riskFactor` is already applied. Through recursion: `StargateStaking` > `StargateAM` > USDC that the `riskFactor` is now applied twice to the LP.

## Impact

`StakedStargateAM` will undervalue tokens. 

## Code Snippet

[AbstractStakingAM.sol#L237-L273](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L237-L273)

## Tool used

Manual Review

## Recommendation

`_calculateValueAndRiskFactors` should be overridden in `StakeStargateAM` to only apply the `riskFactor` tue the reward asset since the underlying LP already has it applied.
