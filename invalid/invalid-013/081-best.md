Oblong Fiery Cheetah

medium

# M-0 Lack of Asset Balance Check in depositInLendingPool Function

## Summary
The absence of a crucial asset balance check in the depositInLendingPool function of the LendingPool contract poses a significant risk of unauthorized transfers..This oversight may disrupt the operational flow of the pool by allowing transfers to occur even when the initiating address lacks sufficient asset balance, potentially leading to failed transactions and operational inefficiencies

## Vulnerability Detail
The absence of a balance check within the depositInLendingPool function permits the execution of asset transfers irrespective of whether the initiating address possesses sufficient assets. This oversight creates a vulnerability wherein unauthorized transactions may occur, potentially compromising the platform's security and exposing users to financial risks.
## Impact
The absence of a balance check in the depositInLendingPool function exposes the platform to multiple attack vectors, including front-running, Sybil attacks, contract logic exploitation, and internal collusion. These vulnerabilities pose risks of unauthorized transfers, fund depletion, financial losses,  and operational disruptions. .

## Code Snippet
[Line 325 in LendingPool.sol](https://github.com/arcadia-finance/lending-v2/blob/main/src/LendingPool.sol#L325)

```javascript

  /**
     * @notice Deposit assets in the Lending Pool.
     * @param assets The amount of assets of the underlying ERC20 tokens being deposited.
     * @param from The address of the Liquidity Provider who deposits the underlying ERC20 token via a Tranche.
     * @dev This function can only be called by Tranches.
     */
    function depositInLendingPool(uint256 assets, address from)
        external
        whenDepositNotPaused
        onlyTranche
        processInterests
    {
        // Need to transfer before minting or ERC777s could reenter.
        // Address(this) is trusted -> no risk on re-entrancy attack after transfer.

        // Absence of balance verification
    // Possibility of unauthorized transfers if 'from' address lacks sufficient balance
        asset.safeTransferFrom(from, address(this), assets);

        unchecked {
            realisedLiquidityOf[msg.sender] += assets;
            totalRealisedLiquidity = SafeCastLib.safeCastTo128(assets + totalRealisedLiquidity);
        }
    }


```

## Tool used

Manual Review

## Recommendation
Implementing a robust balance check within the depositInLendingPool function is recommended to ensure that asset transfers occur only when the initiating address has sufficient assets. Adding a require statement to verify the from address's asset balance before proceeding with transfers will mitigate the risk of unauthorized transactions and enhance platform security.
code:
```diff
function depositInLendingPool(uint256 assets, address from)
    external
    whenDepositNotPaused
    onlyTranche
    processInterests
{
+    require(asset.balanceOf(from) >= assets, "Insufficient balance");
    
    // Proceed with asset transfer after balance check
    asset.safeTransferFrom(from, address(this), assets);

    // Update liquidity and total liquidity after successful transfer
    unchecked {
        realisedLiquidityOf[msg.sender] += assets;
        totalRealisedLiquidity = SafeCastLib.safeCastTo128(assets + totalRealisedLiquidity);
    }
}

```


