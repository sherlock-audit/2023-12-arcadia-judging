Zealous Alabaster Fly

medium

# L2 sequencer down will push an auction's price down, causing unfair liquidation prices, and potentially guaranteeing bad debt

## Summary

The protocol implements a L2 sequencer downtime check in the Registry. In the event of sequencer downtime (as well as a grace period following recovery), liquidations are disabled for the rightful reasons.

However, while the sequencer is down, any ongoing auctions' price decay is still ongoing. When the sequencer goes back online, it will be possible to liquidate for a much lower price, guaranteeing bad debt past a certain point.

## Vulnerability Detail

While the price oracle has sequencer uptime checks, the liquidation auction's price curve calculation does not. The liquidation price is a function with respect to the user's total debt versus their total collateral. 

Due to no sequencer check within the liquidator, the liquidation price continues to decay when the sequencer is down. It is possible for the liquidation price to drop below 100%, that is, it is then possible to liquidate all collateral without repaying all debt. 

Any ongoing liquidations that are temporarily blocked by a sequencer outage will continue to experience price decay. When the sequencer goes back online, liquidation will have dropped significantly in price, causing liquidation to happen at an unfair price as well. Furthermore, longer downtime durations will make it possible to seize all collateral for less than $100\\%$ debt, guaranteeing bad debt for the protocol.

### Proof of concept

We use the [default liquidator parameters defined in the constructor](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Liquidator.sol#L103-L117) for our example:
- Starting multiplier is 150%. 
- Final multiplier is 60%.
- Half-life duration is 1 hour.
- Cutoff time is irrelevant.

Consider the following scenario:

1. Bob's account becomes liquidatable. Someone triggers liquidation start.
2. Anyone can now buy $100\\%$ of Bob's collateral for the price of $150\\%$ of Bob's debt. However, this is not profitable yet, so everyone waits for the price to drop a bit more.
3. After 30 minutes, auction price is now $60\\% + 90\\% * 0.5^{0.5} = 123.63\\%$ of Bob's debt for $100\\%$ collateral. Market price hasn't moved much, so this is still not profitable yet.
4. Sequencer goes down for one hour, not counting grace period. Note that Arbitrum's sequencer has experienced multiple outages of this duration in the past. In 2022, there was an outage of [approx. seven hours](https://cointelegraph.com/news/arbitrum-network-suffers-minor-outage-due-to-hardware-failure). There was also a 78-minute outage [just December 2023](https://cointelegraph.com/news/arbitrum-network-goes-offline-december-15).
6. When the sequencer goes up, the auction has been going on for 1.5 hours, or 1.5 half-lives. Auction price is now $60\\% + 90\\% * 0.5^{1.5} = 91.82\\%$.
7. Liquidation is now profitable. All of Bob's collaterals are liquidated, but the buyer only has to repay $91.82\\%$ of Bob's debt. Bob is left with $0$ collateral but positive debt (specifically, $8.18\\%$ of his original debt).

The impact becomes more severe the longer the sequencer goes down. In addition, the grace period on top of it will decay the auction price even further, before the auction can be back online.
- In the above scenario, if the sequencer outage plus grace period is $2$ hours, then the repaid debt percentage is only $60\\% + 90\\% * 0.5^{2.5} = 75.91\\%$

Furthermore, even if downtime is not enough to bring down the multiplier to less than $100\\%$, Bob will still incur unfair loss due to his collateral being sold at a lower price anyway. Therefore any duration of sequencer downtime will cause an unfair loss.

## Impact

Any ongoing liquidations during a sequencer outage event will execute at a lower debt-to-collateral ratio, potentially guaranteeing bad debt and/or user being liquidated for a lower price.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Liquidator.sol#L364-L395

## Tool used

Manual Review

## Recommendation

Auctions' price curve should either check and exclude sequencer downtime alongside its grace period, or said auctions should simply be voided.
