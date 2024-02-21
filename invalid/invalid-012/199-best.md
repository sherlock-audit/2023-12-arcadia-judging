Early Boysenberry Shetland

medium

# Missing Tranche Validation in donateToTranche Function in LendingPool.sol

## Summary
The donateToTranche function in the provided code snippet lacks a check to ensure the provided trancheIndex corresponds to a valid existing tranche in the tranches array. This vulnerability could lead to unexpected behavior or errors if an invalid index is used.

## Vulnerability Detail
Missing Validation: The function directly accesses the tranches array using the provided trancheIndex without verifying its validity. If trancheIndex is out of bounds (e.g., greater than or equal to the tranches.length), it could lead to:
ArrayIndexOutOfBounds: Attempting to access a non-existent element in the tranches array, potentially causing an exception or unexpected behavior.
Incorrect Donation: Donation being directed to an unintended address or non-existent tranche, potentially losing the donated funds.

## Impact
Potential loss of donated funds due to incorrect target address.
Errors or exceptions during transaction execution.
Difficulty in debugging and identifying the root cause of issues.

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L350

## Tool used

Manual Review

## Recommendation
- Validate Tranche Index: Implement a check before accessing the tranches array. Add a condition like
```solidity
 if (trancheIndex >= tranches.length) revert LendingPoolErrors.TrancheNotFound(); 
```
to ensure the provided index is within the valid range.

Consider Additional Checks: Depending on your specific requirements, you might want to add further checks:
- Verify if the tranche at the given index is active and accepting donations.
- Implement access control mechanisms to restrict donation capabilities to authorized users.

```solidity
function donateToTranche(uint256 trancheIndex, uint256 assets) external whenDepositNotPaused processInterests {
    if (assets == 0) revert LendingPoolErrors.ZeroAmount();

    // Validate tranche index before accessing the array
    if (trancheIndex >= tranches.length) revert LendingPoolErrors.TrancheNotFound();

    address tranche = tranches[trancheIndex];

    // ... (remaining code)
}
```

