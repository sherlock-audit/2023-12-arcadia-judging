Bumpy Concrete Mouse

medium

# Differences in spot price vs AMM prices can be abused to completely misrepresent the holdings of a UniV3 LP tokens

## Summary

`UniswapV3#getPrincipleAmounts` uses the current valuation base on oracles to determine the amount of tokens in a UniV3 LP position. This can be abused by creating an LP position that is outside this price that causes the composition of the token to be entirely different than it actually is. Due to the tiny differences in ticks, this can occur with minute differences between oracle and pool price.

## Vulnerability Detail

[UniswapV3AM.sol#L268-L283](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L268-L283)

    function _getPrincipalAmounts(
        int24 tickLower,
        int24 tickUpper,
        uint128 liquidity,
        uint256 priceToken0,
        uint256 priceToken1
    ) internal pure returns (uint256 amount0, uint256 amount1) {
        // Calculate the square root of the relative rate sqrt(token1/token0) from the trusted USD price of both tokens.
        // sqrtPriceX96 is a binary fixed point number with 96 digits precision.
        uint160 sqrtPriceX96 = _getSqrtPriceX96(priceToken0, priceToken1);

        @audit-issue bases liquidity on oracle prices rather than pool prices
        (amount0, amount1) = LiquidityAmounts.getAmountsForLiquidity(
            sqrtPriceX96, TickMath.getSqrtRatioAtTick(tickLower), TickMath.getSqrtRatioAtTick(tickUpper), liquidity
        );
    }

When calculating the tokens held by an LP position, the `sqrtPriceX96` is used to determine the holdings of the LP token. If this price is outside of the ticks of the LP position it will show the composition of the LP token as either completely `token0` (below) or completely `token1` (above). We can now abuse this to completely misrepresent the holdings of the token.

Assume the following prices for ETH:

Oracle - $1000.01
Pool - $1000

Due to the precision of ticks, pool is a single tick higher than the oracle. Now imagine a position of ETH-USDC that is a single tick position. At the oracle tick, the contract thinks this position would be entirely USDC but at the actual tick of the pool it is entirely ETH. This allows a user to completely bypass exposure limits. As stated in my other submission, these limits are very important for a debt market. 

## Impact

Single tick positions can be used to completely bypass exposure limits

## Code Snippet

[UniswapV3AM.sol#L268-L283](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/UniswapV3/UniswapV3AM.sol#L268-L283)

## Tool used

Manual Review

## Recommendation

This mainly affects LP with extremely narrow spread. There are two potential approaches to fix this. First could be to restrict the spread of the LP token. For example LP shouldn't be allowed if their spread is less than 1000 ticks. The other approach would be to check assets with both the pool price and the oracle price and compare the results. If the assets differ by more than 1% then the transaction should revert.