Gorgeous Opal Mantis

high

# AbstractStakingAM.sol: increaseLiquidity(), mint() will revert expect for first calling, and be failed

## Summary
AbstractStakingAM.sol: increaseLiquidity(), mint() will revert expect for first calling, and be failed, when staking non-standard erc20 token like usdt.

## Vulnerability Detail
AbstractStakingAM.sol: `increaseLiquidity(), mint()` invoke `_stake()` where calls `approve()'.
First approve() set the amount of allowance. 
From second calling of _stake(), some token contracts(such as usdt..) revert the transaction because the previous allowance is not zero.

## Impact
User can't stake more amounts of asset, but just only with first staking

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol#L83
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L285
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L314
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L327
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L351

## Tool used

Manual Review

## Recommendation
Recommend to set the allowance to 0 before calling it.
```solidity
ERC20(asset).approve(address(LP_STAKING_TIME), 0);
ERC20(asset).approve(address(LP_STAKING_TIME), amount);
```