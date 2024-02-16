Acrobatic Brunette Sealion

medium

# LendingPool.startLiquidation() can be called several times for one position

## Summary
```startLiquidation()``` can be called several times for one position in ```LendingPool.sol``` 
## Vulnerability Detail
At each call of the ```startLiquidation()``` ``` initiationReward ``` will be added to ```realisedLiquidityOf```, thus increasing the amount of the reward

```solidity
function startLiquidation(address initiator, uint256 minimumMargin_)
        external
        override
        whenLiquidationNotPaused
        processInterests
        returns (uint256 startDebt)
    {

        // @audit there is no verification that liquidation has begun

        // Only Accounts can have debt, and debtTokens are non-transferrable.
        // Hence by checking that the balance of the msg.sender is not 0,
        // we know that the sender is indeed an Account and has debt.
        startDebt = maxWithdraw(msg.sender);
        if (startDebt == 0) revert LendingPoolErrors.IsNotAnAccountWithDebt();

        // Calculate liquidation incentives which have to be paid by the Account owner and are minted
        // as extra debt to the Account.
        (uint256 initiationReward, uint256 terminationReward, uint256 liquidationPenalty) =
            _calculateRewards(startDebt, minimumMargin_);

        // Mint the liquidation incentives as extra debt towards the Account.
        _deposit(initiationReward + liquidationPenalty + terminationReward, msg.sender);

        // Increase the realised liquidity for the initiator.
        // The other incentives will only be added as realised liquidity for the respective actors
        // after the auction is finished.
        realisedLiquidityOf[initiator] += initiationReward;
        totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);

        // If this is the sole ongoing auction, prevent any deposits and withdrawals in the most jr tranche
        if (auctionsInProgress == 0 && tranches.length > 0) {
            unchecked {
                ITranche(tranches[tranches.length - 1]).setAuctionInProgress(true);
            }
        }

        unchecked {
            ++auctionsInProgress;
        }

        // Emit event
        emit AuctionStarted(msg.sender, address(this), uint128(startDebt));
    }
```


## Impact
Because of the absence of checking whether liquidation has started or not, the function can be called several times for one account, thus attacker increasing own ```initiationReward```

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L861-L901
## Tool used

Manual Review

## Recommendation

add a check for the start of liquidation and use nonReentrant