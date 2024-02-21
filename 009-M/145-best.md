Bumpy Concrete Mouse

medium

# `LendingPool#flashAction` is broken when trying to refinance position across `LendingPools` due to improper access control

## Summary

When refinancing an account, `LendingPool#flashAction` is used to facilitate the transfer. However due to access restrictions on `updateActionTimestampByCreditor`, the call made from the new creditor will revert, blocking any account transfers. This completely breaks refinancing across lenders which is a core functionality of the protocol.

## Vulnerability Detail

[LendingPool.sol#L564-L579](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L564-L579)

    IAccount(account).updateActionTimestampByCreditor();

    asset.safeTransfer(actionTarget, amountBorrowed);

    {
        uint256 accountVersion = IAccount(account).flashActionByCreditor(actionTarget, actionData);
        if (!isValidVersion[accountVersion]) revert LendingPoolErrors.InvalidVersion();
    }

We see above that `account#updateActionTimestampByCreditor` is called before `flashActionByCreditor`.

[AccountV1.sol#L671](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L671)

    function updateActionTimestampByCreditor() external onlyCreditor updateActionTimestamp { }

When we look at this function, it can only be called by the current creditor. When refinancing a position, this function is actually called by the pending creditor since the `flashaction` should originate from there. This will cause the call to revert, making it impossible to refinance across `lendingPools`. 

## Impact

Refinancing is impossible

## Code Snippet

[LendingPool.sol#L529-L586](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L529-L586)

## Tool used

Manual Review

## Recommendation

`Account#updateActionTimestampByCreditor()` should be callable by BOTH the current and pending creditor