Square Pickle Wren

medium

# StakedStargateAM does not implement an emergencyWithdraw() function

## Summary

## Vulnerability Detail
The [`LP_STAKING_TIME`](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#code) contract of stargate finance deployed on base implements an `emergencyWithdraw()` function that allows to withdraw LPs from the staking contract ignoring rewards. 

This is because in the [`LP_STAKING_TIME`](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#code) are withdrawn automatically on every action (deposit, withdraw) and the contract might not contain enough rewards for the transfer to succeed, which will make the withdrawal/deposit revert.

Because of this [StakedStargateAM](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol#L16) might be unable to withdraw LPs from [`LP_STAKING_TIME`](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#code), leaving the funds stucked.

## Impact
Funds can get stuck, this can be fixed by manually sending rewards tokens to the [`LP_STAKING_TIME`](https://basescan.org/address/0x06Eb48763f117c7Be887296CDcdfad2E4092739C#code) contract, given that the rewards tokens are easily accessible through the free market. Since it can be fixed by sending tokens directly to the `LP_STAKING_TIME` contract I think this is a low/QA issue, but I'm not sure so I'm submitting it as medium in case having to send tokens to the `LP_STAKING_TIME` contract directly is considered a loss of funds.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Implement a `emergencyWithdraw()` function in [StakedStargateAM](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StakedStargateAM.sol) that allows to withdraw funds ignoring the rewards amount.
