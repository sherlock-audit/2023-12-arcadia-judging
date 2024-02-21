Powerful Myrtle Hawk

medium

# Lack of `minAnswer` check for oracle may lead to bad pricing during flash crashes

## Summary

Chainlink price feeds have in-built minimum & maximum prices they will return. When the `Black Swan Event` happens, and a flash crash happens(like LUNA Coin), even when the price of an asset falls below the price feed’s minimum price, only the minimum price is returned. So if the `minAnswer` check is not performed, the actual `answer` get is incorrect. This could lead to a very bad situation due to bad pricing. For example, the account will remain `healthy` which should be liquidated.

## Vulnerability Detail

In [ChainlinkOM::_getLatestAnswer](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/oracle-modules/ChainlinkOM.sol#L118-L128), the `answer_` is not compared with `minAnswer` and `maxAnswer` which should be queried from `Aggregator`.

```solidity
        try IChainLinkData(oracleInformation_.oracle).latestRoundData() returns (
            uint80 roundId, int256 answer_, uint256, uint256 updatedAt, uint80
        ) {
            if (
                roundId > 0 && answer_ >= 0 && updatedAt > block.timestamp - oracleInformation_.cutOffTime
                    && updatedAt <= block.timestamp
            ) {
                success = true;
                answer = uint256(answer_);
            }
        } catch { }
```

The problem is that Chainlink price feeds have in-built minimum & maximum prices they will return. When the `Black Swan Event` happens, and a flash crash happens(like LUNA Coin), even when the price of an asset falls below the price feed’s minimum price, only the minimum price is returned.

Since this contract uses Oracle to calculate the price of an asset, this will lead to the following situation:
1. A coin has experienced a flash crash, and its price goes down to $0.05.
2. The `minAnswer` in the `Aggregator` is set to be `$0.1` and is not updated.
3. The returned value of the price will be much higher than what it should be.

As a consequence, an account could remain `healthy` which should be liquidated. The user could buy the coin at market price and deposit it into the protocol to avoid liquidation.

## Impact

Lack of `minAnswer` check could lead to bad pricing.  As a consequence, an account could remain `healthy` which should be liquidated. And user could buy the coin at market price and deposit it into the protocol to avoid liquidation. The liquidation mechanism will be greatly affected.

## Code Snippet

[ChainlinkOM::_getLatestAnswer](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/oracle-modules/ChainlinkOM.sol#L118-L128)
```solidity
        try IChainLinkData(oracleInformation_.oracle).latestRoundData() returns (
            uint80 roundId, int256 answer_, uint256, uint256 updatedAt, uint80
        ) {
            if (
                roundId > 0 && answer_ >= 0 && updatedAt > block.timestamp - oracleInformation_.cutOffTime
                    && updatedAt <= block.timestamp
            ) {
                success = true;
                answer = uint256(answer_);
            }
        } catch { }
```

## Tool used

Manual Review, VScode

## Recommendation

Add a check that the `answer` falls in the range of `minAnswer` and `maxAnswer` so that the price is valid.