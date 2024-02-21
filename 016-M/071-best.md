Zealous Alabaster Fly

medium

# Liquidation may not be profitable until bad debt is guaranteed

## Summary

When an account is liquidatable, anyone can start a liquidation auction for the account. The auction price function is an exponential decay curve with respect to time, and is valued by the total value of the debt, as well as the max/min price multiplier.

We argue that, under extreme market conditions and reasonable market configs, liquidation may not be profitable until it's possible to buy all collateral for less than 100% debt, guaranteeing bad debt for the protocol.

## Vulnerability Detail

The auction price curve is a time-based decay function of the total debt to total collateral ratio. The ratio starts at `startPriceMultiplier`, exponentially decaying down towards `minPriceMultiplier`, ending at auction cutoff time. 
- For example, if `startPriceMultiplier` is set at 150%, then at auction start, it is possible to seize 100% of collateral by repaying 150% of the account's total debt. The percentage goes down with time.

To demonstrate how the auction price curve can make a liquidation unprofitable until bad debt, we take the following risk parameters as an example:
- Let the collateral asset be (W)ETH, and the borrowing asset be USDC.
- Let the liquidation factor for ETH be $85\\%$. 
  - This is lower than the example provided in the [arcadia docs](https://docs.arcadia.finance/protocol/arcadia-accounts).
- Let `startPriceMultiplier` be $150\\%$, [as defined in the contract constructor](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Liquidator.sol#L103-L117).
- Let the half-life be 1 hour, for the same reason as above.

Assuming ETH current price is \$1000. That means suppose Bob has $1$ ETH as collateral, he may have a maximum debt of $850$ USDC. Let's say Bob is liquidated right as this point:
- At time $0$, `startPriceMultiplier` is $1.5$. That means the liquidation price is $850 * 1.5 = 1275$ USDC for $1$ ETH. This is clearly unprofitable.
- The liquidation becomes profitable when the price multiplier decays down to $1000$ for $1$ ETH. The price multiplier hence has to drop down to $1.17$. The duration required for this is $log_{2}(\frac{150-60}{117-60}) = 0.66$ half-lives, or $40$ minutes.

The problem arises when ETH price drops sharply during these $40$ minutes. Any drop in price will push the $40$ minutes mark further, and the auction will require a deeper price decay for the liquidation to actually become profitable.

The auction will generate bad debt when the price multiplier drops to $1$ or lower. The time duration required is $log_{2}(\frac{150-60}{100-60}) = 1.17$ half-lives, or $70$ minutes. Then during extreme market conditions, if ETH price drops to below \$850 after $70$ minutes of auction, liquidation will only be profitable at lower than $100\\%$ account total debt, causing bad debt for the protocol by definition.

This may seem like a hypothetical scenario, however cryptocurrency itself is a highly volatile asset, and all it takes to guarantee an unhappy liquidation is a flash crash. Replacing WETH and USDC with two other assets, and opposite direction price movements are likely.

Furthermore, liquidation and repayment of two different assets will involve swapping on external exchanges to convert the collateral to the repayment asset, and this effect actually stacks when there are multiple accounts being liquidated, as the external market has to carry the sell pressure for all these liquidations. Therefore in such a market condition, a profitable liquidation may still not be achievable when the market price hits the required price for healthy liquidation.

All in all, while the admin are able to set price curve parameters to the liquidator, the price curve is set for all of the liquidator's associated markets. The price curve is fundamentally a risk factor by nature, therefore a single one will not be able to reflect the risk associated with all supported assets - each supported asset/creditor should have its own custom auction curve, rather than an all-in-one curve for all assets.
- The end result is that either profitable liquidation may not be possible on some assets, while users may be liquidated on an unfair price curve on other assets.

## Impact

Liquidation curve math may prevent profitable liquidation until an account becomes insolvent (collateral < borrows), creating bad debt for the protocol.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Liquidator.sol#L103-L116

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Liquidator.sol#L364-L395

## Tool used

Manual Review

## Recommendation

The auction curve parameters are fundamentally risk factors by nature, and should not be set as global parameters for all assets for a single liquidator. Each asset has different quality and volatility, thereby the liquidation curve should be set to reflect such risks.

Some mitigation methods include:
- Set the liquidation curve math as a risk factor for each separate asset or creditor, instead of being the liquidator's parameter applying for all its associated creditors.
- Set the liquidation curve as a function of the collateral value, debt value, and risk factors of the lending asset itself, as opposed to just the debt amount.

Ideally one would want a liquidation to be immediately profitable from the start, or at least zero-sum as a dutch auction starting price. Liquidation is the only protection against insolvency, therefore there should not be a flow where liquidation is never profitable.
