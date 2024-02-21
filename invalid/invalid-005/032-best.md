Powerful Slate Hawk

medium

# No more than 256 tranches to a single lending pool

## Summary
According to [docs](https://audits.sherlock.xyz/contests/137) Invariants - LendingPool

> No more than 256 tranches to a single lending pool.

## Vulnerability Detail
There is no restriction to that in the code or maybe due to silent cast `uint8`
```solidity
    function addTranche(address tranche, uint16 interestWeight_) external onlyOwner processInterests {
        if (auctionsInProgress > 0) revert LendingPoolErrors.AuctionOngoing();
        if (isTranche[tranche]) revert LendingPoolErrors.TrancheAlreadyExists();

        totalInterestWeight += interestWeight_;
        interestWeightTranches.push(interestWeight_);
        interestWeight[tranche] = interestWeight_;

        uint8 trancheIndex = uint8(tranches.length);
        tranches.push(tranche);
        isTranche[tranche] = true;

        emit InterestWeightTrancheUpdated(tranche, trancheIndex, interestWeight_);
    }

```
[LendingPool.sol#L216](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L216)
## Impact

## Code Snippet
POC - insert in `AddTranche.fuzz.t.sol`

```solidity
    function testFuzz_Success_add_300_tranche(uint16 interestWeight) public {
        vm.startPrank(users.creatorAddress);
        for (uint16 i; i < 300; i++) {
            tranche = new TrancheExtension(address(pool), 111, "Tranche", "T");
            pool_.addTranche(address(tranche), i + 1);
        }

        assertTrue(pool_.numberOfTranches() < 256);
    }

```
## Tool used

Manual Review

## Recommendation
```diff
    function addTranche(address tranche, uint16 interestWeight_) external onlyOwner processInterests {
        if (auctionsInProgress > 0) revert LendingPoolErrors.AuctionOngoing();
        if (isTranche[tranche]) revert LendingPoolErrors.TrancheAlreadyExists();

        totalInterestWeight += interestWeight_;
        interestWeightTranches.push(interestWeight_);
        interestWeight[tranche] = interestWeight_;

-        uint8 trancheIndex = uint8(tranches.length);
+        uint8 trancheIndex = SafeCastLib.safeCastTo8(tranches.length);
        tranches.push(tranche);
        isTranche[tranche] = true;

        emit InterestWeightTrancheUpdated(tranche, trancheIndex, interestWeight_);
    }
```
