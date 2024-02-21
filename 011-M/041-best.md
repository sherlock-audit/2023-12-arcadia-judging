Square Pickle Wren

high

# Precision loss in the valuation of UniswapV3 positions might lead to premature liquidations

## Summary
A precision loss in the valuation of UniswapV3 positions might lead to premature liquidations causing unexpected loss to the user.

## Vulnerability Detail
The function [_getPrincipalAmounts()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L268) in UniswapV3AM calculates the underlying token amounts of a UniswapV3 liquidity position.  
Internally it calls the function [_getSqrtPriceX96()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L297) which is responsible for calculating the square root of the relative price of two tokens with 96 bits of precision:
```solidity
function _getSqrtPriceX96(uint256 priceToken0, uint256 priceToken1) internal pure returns (uint160 sqrtPriceX96) {
    ...
    uint256 priceXd18 = priceToken0.mulDivDown(1e18, priceToken1); 
    uint256 sqrtPriceXd9 = FixedPointMathLib.sqrt(priceXd18);
    sqrtPriceX96 = uint160((sqrtPriceXd9 << FixedPoint96.RESOLUTION) / 1e9);
}
```

The vulnerability lies in the calculation of `priceXd18` which is not done with enough precision. If `priceToken1` is bigger than `priceToken0 * 1e18`, `priceXd18` will be set to 0. The implication is that `sqrtPriceX96` will also be `0`, which will result in [_getPrincipalAmounts()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L268) believing the whole liquidity position is constituted of token0 and the protocol evaluating the position incorrectly.

Let's suppose we have a UniswapV3 position of two tokens:
- token0: SHIB, Value: ~$0.000009, Decimals: 18
- token1: WBTC, Value: ~$40.000, Decimals: 8

The protocol evaluates all token prices for 1e18 units of token:
- `priceToken0`: ((0.000009*1e18)*1e18)/1e18 = 9e12
- `priceToken1`: ((40000*1e18) * 1e18)/1e8 = 4e32
- `priceXd18`: (9e12 * 1e18)/4e32 = 0
- `sqrtPriceX96`: 0

The `0` value will be then passed to [getAmountsForLiquidity()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/libraries/LiquidityAmounts.sol#L63-L64) as `sqrtRatioX96` which will evaluate the position as if it only contains token0 (SHIB) tokens:
```solidity
if (sqrtRatioX96 <= sqrtRatioAX96) {
    amount0 = getAmount0ForLiquidity(sqrtRatioAX96, sqrtRatioBX96, liquidity);
}
```
Because of this some uniswapV3 positions of tokens with a big discrepancy in price and decimals will be evaluated incorrectly. This can happen while the position is already deposited because of price fluctuations. 

### POC
Here's a runnable POC that can be copy-pasted in `GetUnderlyingAssetsAmounts.fuzz.t.sol`. It shows that with a token0 with 18 decimals valued $1 and a token1 with 2 decimals valued $1000 dollars there is a precision loss that results in `sqrtRatioX96` being 0:

```solidity
function test_UniV3SqrtPrecisionLoss() public {
    UnderlyingAssetState memory asset0 = UnderlyingAssetState({decimals: 18, usdValue: 1}); //A token with 18 decimals valued 1$
    UnderlyingAssetState memory asset1 = UnderlyingAssetState({decimals: 2, usdValue: 1000}); //A token with 2 decimals valued 1000$

    uint96 tokenId = 197964524549228351;

    ERC20Mock token1 = new ERC20Mock("Token 1", "TOK1", uint8(asset1.decimals));
    ERC20Mock token0 = new ERC20Mock("Token 0", "TOK0", uint8(asset0.decimals));

    if (token0 > token1) {
        (token0, token1) = (token1, token0);
        (asset0, asset1) = (asset1, asset0);
    }
    NonfungiblePositionManagerMock.Position memory position = NonfungiblePositionManagerMock.Position({
        nonce: 0,
        operator: address(0),
        poolId: 1,
        tickLower: -887272,
        tickUpper: 887272,
        liquidity: 1e18,
        feeGrowthInside0LastX128: 0,
        feeGrowthInside1LastX128: 0,
        tokensOwed0: 0,
        tokensOwed1: 0
    });

    addUnderlyingTokenToArcadia(address(token0), int256(asset0.usdValue));
    addUnderlyingTokenToArcadia(address(token1), int256(asset1.usdValue));
    IUniswapV3PoolExtension pool = createPool(token0, token1, 1e18, 300);
    nonfungiblePositionManagerMock.setPosition(address(pool), tokenId, position);

    uint256[] memory underlyingAssetsAmounts;
    AssetValueAndRiskFactors[] memory rateUnderlyingAssetsToUsd;
    {
        bytes32 assetKey = bytes32(abi.encodePacked(tokenId, address(nonfungiblePositionManagerMock)));
        (underlyingAssetsAmounts, rateUnderlyingAssetsToUsd) =
            uniV3AssetModule.getUnderlyingAssetsAmounts(address(creditorUsd), assetKey, 1, new bytes32[](0));
    }

    uint256 expectedRateUnderlyingAssetsToUsd0 = asset0.usdValue * 10 ** (36 - asset0.decimals);
    uint256 expectedRateUnderlyingAssetsToUsd1 = asset1.usdValue * 10 ** (36 - asset1.decimals);
    assertEq(rateUnderlyingAssetsToUsd[0].assetValue, expectedRateUnderlyingAssetsToUsd0);
    assertEq(rateUnderlyingAssetsToUsd[1].assetValue, expectedRateUnderlyingAssetsToUsd1);

    uint160 sqrtPriceX96 =
        uniV3AssetModule.getSqrtPriceX96(expectedRateUnderlyingAssetsToUsd0, expectedRateUnderlyingAssetsToUsd1);

    //❌ sqrtPriceX96 is equal to zero because of precision loss
    assertEq(sqrtPriceX96, 0); 

    (uint256 expectedUnderlyingAssetsAmount0, uint256 expectedUnderlyingAssetsAmount1) = LiquidityAmounts
        .getAmountsForLiquidity(
        sqrtPriceX96,
        TickMath.getSqrtRatioAtTick(position.tickLower),
        TickMath.getSqrtRatioAtTick(position.tickUpper),
        position.liquidity
    );

    //❌ The position is considered to be constitued only by `token0` 
    assertGt(expectedUnderlyingAssetsAmount0, 0); 
    assertEq(expectedUnderlyingAssetsAmount1, 0);
}
```

## Impact
UniswapV3 positions are evaluated incorrectly, which might lead to premature liquidations (ie. unexpected loss of funds for the user). 

## Code Snippet

## Tool used

Manual Review

## Recommendation
This is a "division before multiplication" issue: the function first divides by `priceToken1` and then multiplies by `2**96` (`sqrtPriceXd9 << FixedPoint96.RESOLUTION`). Multiply by `2^96` before diving by `priceToken1` while taking care of possible overflows and adjusting the decimals after the square root by multiplying for `2^48`.
