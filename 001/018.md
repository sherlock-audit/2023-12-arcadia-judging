Powerful Slate Hawk

medium

# reward tokens will be stuck in staking contract

## Summary
Some amount of tokens will be stuck in a staking contract after users burn their positions. POC below
## Vulnerability Detail
The `_getRewardBalances` function calculations are off. The problem arises after two consecutive mints, on the third interaction with a stakingmodule. An amount of rewards equal to the pending rewards at the second interaction are then claimed but not added to the reward balances.
```solidity
    function _getRewardBalances(AssetState memory assetState_, PositionState memory positionState_)
        internal
        view
        returns (AssetState memory, PositionState memory)
    {
        if (assetState_.totalStaked > 0) {
            // Calculate the new assetState
            // Fetch the current reward balance from the staking contract.
            uint256 currentRewardGlobal = _getCurrentReward(positionState_.asset);
            // Calculate the increase in rewards since last Asset interaction.
            uint256 deltaReward = currentRewardGlobal - assetState_.lastRewardGlobal;
            uint256 deltaRewardPerToken = deltaReward.mulDivDown(1e18, assetState_.totalStaked);
            // Calculate and update the new RewardPerToken of the asset.
            // unchecked: RewardPerToken can overflow, what matters is the delta in RewardPerToken between two interactions.
            unchecked {
                assetState_.lastRewardPerTokenGlobal =
                    assetState_.lastRewardPerTokenGlobal + SafeCastLib.safeCastTo128(deltaRewardPerToken);
            }
            // Update the reward balance of the asset.
            assetState_.lastRewardGlobal = SafeCastLib.safeCastTo128(currentRewardGlobal);

            // Calculate the new positionState.
            // Calculate the difference in rewardPerToken since the last position interaction.
            // unchecked: RewardPerToken can underflow, what matters is the delta in RewardPerToken between two interactions.
            unchecked {
                deltaRewardPerToken = assetState_.lastRewardPerTokenGlobal - positionState_.lastRewardPerTokenPosition;
            }
            // Calculate the rewards earned by the position since its last interaction.
            // unchecked: deltaRewardPerToken and positionState_.amountStaked are smaller than type(uint128).max.
            unchecked {
                deltaReward = deltaRewardPerToken * positionState_.amountStaked / 1e18;
            }
            // Update the reward balance of the position.
            positionState_.lastRewardPosition =
                SafeCastLib.safeCastTo128(positionState_.lastRewardPosition + deltaReward);
        }
        // Update the RewardPerToken of the position.
        positionState_.lastRewardPerTokenPosition = assetState_.lastRewardPerTokenGlobal;

        return (assetState_, positionState_);
    }


```
[AbstractStakingAM.sol#L529](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/abstracts/AbstractStakingAM.sol#L529)
## Impact

## Code Snippet

## Tool used
POC
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.22;

import "forge-std/Test.sol";
import {StakedStargateAM, IRegistry, ILpStakingTime} from "../src/asset-modules/Stargate-Finance/StakedStargateAM.sol";


contract Registry {
    function addAsset(address asset) public {
    }

    function isAllowed(address asset, uint256 assetId) public view returns (bool){
        return true;
    }
}
interface IERC20 {
    event Approval(address indexed owner, address indexed spender, uint256 value);
    event Transfer(address indexed from, address indexed to, uint256 value);

    function name() external view returns (string memory);

    function symbol() external view returns (string memory);

    function decimals() external view returns (uint8);

    function totalSupply() external view returns (uint256);

    function balanceOf(address owner) external view returns (uint256);

    function allowance(address owner, address spender) external view returns (uint256);

    function approve(address spender, uint256 value) external returns (bool);

    function transfer(address to, uint256 value) external returns (bool);

    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function withdraw(uint256 wad) external;
    function deposit(uint256 wad) external returns (bool);
    function owner() external view returns (address);
}

contract RewardsStuck is Test {
    IERC20 USDbCpool = IERC20(0x4c80E24119CFB836cdF0a6b53dc23F04F7e652CA);
    IERC20 rewardToken = IERC20(0xE3B53AF74a4BF62Ae5511055290838050bf764Df);
    ILpStakingTime LpStakingTime = ILpStakingTime(0x06Eb48763f117c7Be887296CDcdfad2E4092739C);

    StakedStargateAM stakedStargateAM;
    Registry registry;
    address user1 = 0x160B6772c9976d21ddFB3e3211989Fa099451af7;
    address user2 = 0x2db0500e1942626944efB106D6A66755802Cef20;

    function setUp() public {
        vm.createSelectFork("https://mainnet.base.org", 10_116_031);

        registry = new Registry();
    }

    function test() external {
        stakedStargateAM = new StakedStargateAM(address(registry), address(LpStakingTime));
        stakedStargateAM.addAsset(1);
        deal(address(USDbCpool), address(this), 10000);
        deal(address(USDbCpool), address(user1), 10000);
        USDbCpool.approve(address(stakedStargateAM), type(uint256).max);

        uint positionId = stakedStargateAM.mint(address(USDbCpool), 10000);
        console.log("------------------");

        vm.warp(block.timestamp + 1000);

        vm.startPrank(address(user1));
        USDbCpool.approve(address(stakedStargateAM), type(uint256).max);
        uint positionId2 = stakedStargateAM.mint(address(USDbCpool), 10000);

        vm.warp(block.timestamp + 1000);
        console.log("staked in the middle: ", stakedStargateAM.totalStaked(address(USDbCpool)));

        vm.stopPrank();
        stakedStargateAM.burn(positionId);

        vm.warp(block.timestamp + 1000);

        vm.startPrank(address(user1));
        stakedStargateAM.burn(positionId2);
        console.log("--------after buring all----------");

        console.log("rewardToken.balanceOf(address(stakedStargateAM)): ", rewardToken.balanceOf(address(stakedStargateAM)));
        console.log("rewardToken.balanceOf(address(address(this))): ", rewardToken.balanceOf(address(this)));
        console.log("rewardToken.balanceOf(address(user1)): ", rewardToken.balanceOf(address(user1)));
        console.log("stakedStargateAM.totalStaked(address(USDbCpool)) ", stakedStargateAM.totalStaked(address(USDbCpool)));
    }

    function onERC721Received(address, address, uint256, bytes memory) external returns (bytes4) {
        return this.onERC721Received.selector;
    }
}

```

```code
  ------------------
  staked in the middle:  20000
  --------after buring all----------
  rewardToken.balanceOf(address(stakedStargateAM)):  48055843493
  rewardToken.balanceOf(address(address(this))):  72083765165
  rewardToken.balanceOf(address(user1)):  72083765165
  stakedStargateAM.totalStaked(address(USDbCpool))  0
```
Manual Review

## Recommendation
Either create a function so the owner can sweep the rest of the tokens, or change formula so there will be no left tokens left, or give all the rest reward tokens to the last staker.