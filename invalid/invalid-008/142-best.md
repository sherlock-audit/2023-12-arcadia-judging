Bumpy Concrete Mouse

medium

# Users can easily bypass exposure limits by adding liquidity to UniV3 LP tokens after deposit

## Summary

Exposure is calculated when the UniV3 token is deposited into the contract and isn't ever recalculated until it is withdrawn. The problem with this is that UniV3 tokens can have liquidity added permissionlessly. This allows a user to deposit a token then add large amounts of liquidity after and completely bypass the exposure limit.

## Vulnerability Detail

[NonfungiblePositionManager.sol#L198-L203](https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/NonfungiblePositionManager.sol#L198-L203)

    function increaseLiquidity(IncreaseLiquidityParams calldata params)
        external
        payable
        override
        checkDeadline(params.deadline)
        returns (

Notice above how `NonfungiblePositionManager#increaseLiquidity` is missing the `isAuthorizedForToken` modifier. This means that any account can add liquidity to a position at anytime regardless of who the owner is.

[AccountV1.sol#L843-L844](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L843-L844)

    uint256[] memory assetTypes =
        IRegistry(registry).batchProcessDeposit(creditor, assetAddresses, assetIds, assetAmounts);

When depositing an asset, `registry#batchProcessDeposit` is called. This will value the token based on the current composition of the token and apply that value against the exposure limit.

[LendingPool.sol#L414-L454](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L414-L454)

    function borrow(uint256 amount, address account, address to, bytes3 referrer)
        external
        whenBorrowNotPaused
        processInterests
    {
        address accountOwner = IFactory(ACCOUNT_FACTORY).ownerOfAccount(account);
        if (accountOwner == address(0)) revert LendingPoolErrors.IsNotAnAccount();

        uint256 amountWithFee = amount + amount.mulDivUp(originationFee, ONE_4);

        if (accountOwner != msg.sender) {
            ...
        }

        _deposit(amountWithFee, account);

        unchecked {
            if (amountWithFee - amount > 0) {
                totalRealisedLiquidity = SafeCastLib.safeCastTo128(amountWithFee + totalRealisedLiquidity - amount);
                realisedLiquidityOf[treasury] += amountWithFee - amount;
            }
        }
        uint256 accountVersion = IAccount(account).increaseOpenPosition(maxWithdraw(account));
        if (!isValidVersion[accountVersion]) revert LendingPoolErrors.InvalidVersion();

        asset.safeTransfer(to, amount);

        emit Borrow(account, msg.sender, to, amount, amountWithFee - amount, referrer);
    }


As seen above from `LendingPool#borrow`, the exposure of the asset is never recomputed. This enables adding a large amount of liquidity to the UniV3 LP token that is never accounted for by exposure limits. The impact of bypassing these exposure limits should not be underestimated. As a good example, AAVE has Gauntlet which uses large and costly models to determine the risk factors of the market to always ensure it's solvency. Bypassing these limits with such magnitude is very dangerous for the stability and solvency of the market.

## Impact

Exposure limits can be bypassed completely, allowing for markets to take on much more collateral than intended.

## Code Snippet

[AccountV1.sol#L681-L697](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L681-L697)

## Tool used

Manual Review

## Recommendation

Whenever a position borrows against a UniV3 LP token, indirect deposit should be called to update the valuation of the token.