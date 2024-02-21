Square Marigold Aardvark

high

# Did not set a value to the `interestWeight` variable in the `LendingPool.sol#setTreasuryWeights` function.

## Summary
Because the value is not set to the `interestWeight` variable in the `LendingPool.sol#setTreasuryWeights` function, if `owner_` is `treasury` and `lastSyncedTimestamp != block.timestamp` in the `LendingPool.sol#liquidityOf` function, the return value is It is inaccurate.
## Vulnerability Detail
The `LendingPool.sol#setTreasuryWeights` function is as follows.
```solidity
function setTreasuryWeights(uint16 interestWeight_, uint16 liquidationWeight) external onlyOwner processInterests {
290:        totalInterestWeight = totalInterestWeight - interestWeightTreasury + interestWeight_;
291:
292:        emit TreasuryWeightsUpdated(
293:            interestWeightTreasury = interestWeight_, liquidationWeightTreasury = liquidationWeight
294:        );
}
```
As you can see on the right, the `LendingPool.sol#setTreasuryWeights` function does not set a value for `treasury` in the `interestWeight` variable.
And the `LendingPool.sol#liquidityOf` function is as follows.
```solidity
function liquidityOf(address owner_) external view returns (uint256 assets) {
641:        // Avoid a second calculation of unrealised debt (expensive).
642:        // if interests are already synced this block.
643:        if (lastSyncedTimestamp != uint32(block.timestamp)) {
644:            // The total liquidity of a tranche equals the sum of the realised liquidity
645:            // of the tranche, and its pending interests.
646:            uint256 interest = calcUnrealisedDebt().mulDivDown(interestWeight[owner_], totalInterestWeight);
647:            unchecked {
648:                assets = realisedLiquidityOf[owner_] + interest;
649:            }
650:        } else {
651:            assets = realisedLiquidityOf[owner_];
652:        }
    }
```
In `L646`, when `owner_=treasury`, `interestWeight[owner_]=0`, so `interest=0` is established.
In other words, the `LendingPool.sol#liquidityOf` function does not accurately return the liquidity of the treasury.
## Impact
Treasury liquidity cannot be obtained accurately.
## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L289C1-L295C6
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L640C5-L653C6
## Tool used

Manual Review

## Recommendation
Modify the `LendingPool.sol#setTreasuryWeights` function as follows.
```solidity
function setTreasuryWeights(uint16 interestWeight_, uint16 liquidationWeight) external onlyOwner processInterests {
        totalInterestWeight = totalInterestWeight - interestWeightTreasury + interestWeight_;
++      interestWeight[treasury] = interestWeight_;

        emit TreasuryWeightsUpdated(
            interestWeightTreasury = interestWeight_, liquidationWeightTreasury = liquidationWeight
        );
    }
```
