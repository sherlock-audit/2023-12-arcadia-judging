Square Pickle Wren

high

# Stargate `STG` rewards are accounted incorrectly by `StakedStargateAM.sol`

## Summary
Stargate [LP_STAKING_TIME](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#code) contract clears and sends rewards to the caller every time `deposit()` is called but [StakedStargateAM](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol) does not take it into account.

## Vulnerability Detail
 When either [mint()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L285-L320) or [increaseLiquidity()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L327-L354) are called the `assetState[asset].lastRewardGlobal` variable is not reset to `0` even though the rewards have been transferred and accounted for on stargate side.

After a call to [mint()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L285-L320) or [increaseLiquidity()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L327-L354) any subsequent call to either [mint()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L285-L320), [increaseLiquidity()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L327-L354), [burn()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L360), [decreaseLiquidity()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L371), [claimRewards()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L430) or [rewardOf()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L511-L520), which all internally call [_getRewardBalances()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L529-L569),  will either revert for underflow or account for less rewards than it should because `assetState_.lastRewardGlobal` has not been correctly reset to `0` but `currentRewardGlobal` (which is fetched from stargate) has:
```solidity
uint256 currentRewardGlobal = _getCurrentReward(positionState_.asset);
uint256 deltaReward = currentRewardGlobal - assetState_.lastRewardGlobal; ❌
```
```solidity
function _getCurrentReward(address asset) internal view override returns (uint256 currentReward) {
    currentReward = LP_STAKING_TIME.pendingEmissionToken(assetToPid[asset], address(this));
}
```
### POC
To copy-paste in `USDbCPool.fork.t.sol`:
```solidity
function testFork_WrongRewards() public {
    uint256 initBalance = 1000 * 10 ** USDbC.decimals();
    // Given : A user deposits in the Stargate USDbC pool, in exchange of an LP token.
    vm.startPrank(users.accountOwner);
    deal(address(USDbC), users.accountOwner, initBalance);

    USDbC.approve(address(router), initBalance);
    router.addLiquidity(poolId, initBalance, users.accountOwner);
    // assert(ERC20(address(pool)).balanceOf(users.accountOwner) > 0);

    // And : The user stakes the LP token via the StargateAssetModule
    uint256 stakedAmount = ERC20(address(pool)).balanceOf(users.accountOwner);
    ERC20(address(pool)).approve(address(stakedStargateAM), stakedAmount);
    uint256 tokenId = stakedStargateAM.mint(address(pool), uint128(stakedAmount) / 4);

    //We let 10 days pass to accumulate rewards.
    vm.warp(block.timestamp + 10 days);

    // User increases liquidity of the position.
    uint256 initialRewards = stakedStargateAM.rewardOf(tokenId);
    stakedStargateAM.increaseLiquidity(tokenId, 1);

    vm.expectRevert();
    stakedStargateAM.burn(tokenId); //❌ User can't call burn because of underflow

    //We let 10 days pass, this accumulates enough rewards for the call to burn to succeed
    vm.warp(block.timestamp + 10 days);
    uint256 currentRewards = stakedStargateAM.rewardOf(tokenId);
    stakedStargateAM.burn(tokenId);

    assert(currentRewards - initialRewards < 1e10); //❌ User gets less rewards than he should. The rewards of the 10 days the user couldn't withdraw his position are basically zeroed out.
    vm.stopPrank();
}
```


## Impact

Users will not be able to take any action on their positions until `currentRewardGlobal` is greater or equal to `assetState_.lastRewardGlobal`. After that they will be able to perform actions but their position will account for less rewards than it should because a total amount of `assetState_.lastRewardGlobal` rewards is nullified. 

This will also DOS the whole lending/borrowing system if an Arcadia Stargate position is used as collateral because [rewardOf()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L511-L520), which is called to estimate the collateral value, also reverts.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Adjust the `assetState[asset].lastRewardGlobal` correctly or since every action (`mint()`, `burn()`, `increaseLiquidity()`, `decreaseliquidity()`, `claimReward()`) will have the effect of withdrawing all the current rewards it's possible to change the function [_getRewardBalances()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L529-L569) to use the amount returned by [_getCurrentReward()](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol#L113-L115) as the `deltaReward` directly:
```solidity
uint256 deltaReward = _getCurrentReward(positionState_.asset);
```


