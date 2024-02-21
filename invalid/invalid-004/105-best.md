Salty Lead Fox

medium

# `Registry::getRateInUsd()` does not check whether L2 sequencer is up

## Summary
`Registry::getRateInUsd()` does not apply a check for whether the L2 sequencer is up. This will cause stale prices to be returned when it's down.

## Vulnerability Detail
`Registry::getRateInUsd()` interacts with the `ChainlinkOM` contract to get rates of primary assets in USD. The problem here is that neither `ChainlinkOM` nor `Registry::getRateInUsd()` check if the L2 sequencer is up. This will cause stale prices to be returned when it's down. This affects the `Registry` and `ERC20PrimaryAM` contracts, which in turn will affect all accounts interacting with those.

```solidity
function getRateInUsd(bytes32 oracleSequence) external view returns (uint256 rate) {
    (bool[] memory baseToQuoteAsset, uint256[] memory oracles) = oracleSequence.unpack();

    rate = 1e18;

    uint256 length = oracles.length;
    for (uint256 i; i < length; ++i) {
        // Each Oracle has a fixed base asset and quote asset.
        // The oracle-rate expresses how much tokens of the quote asset (18 decimals precision) are required
        // to buy 1 token of the BaseAsset.
        if (baseToQuoteAsset[i]) {
            // "Normal direction" (how much of the QuoteAsset is required to buy 1 token of the BaseAsset).
            // -> Multiply with the oracle-rate.
            rate = rate.mulDivDown(IOracleModule(oracleToOracleModule[oracles[i]]).getRate(oracles[i]), 1e18);
        } else {
            // "Inverse direction" (how much of the BaseAsset is required to buy 1 token of the QuoteAsset).
            // -> Divide by the oracle-rate.
            rate = rate.mulDivDown(1e18, IOracleModule(oracleToOracleModule[oracles[i]]).getRate(oracles[i]));
        }
    }
}
```

## Impact
Data returned from `Registry::getRateInUsd()` could be stale if the sequencer goes down. Affected will be `Registry`, `ERC20PrimaryAM`, the accounts, as well as any other external callers.

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/Registry.sol#L580-L600
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/oracle-modules/ChainlinkOM.sol#L141-L157

## Tool used

Manual Review

## Recommendation
Apply the already available `sequencerNotDown` modifier to `Registry::getRateInUsd()` as well.
