Bumpy Concrete Mouse

medium

# `COOL_DOWN_PERIOD` is not long enough to prevent block stuffing on some L2s

## Summary

`AccountV1` automatically locks ownership whenever the value of the account is affected (i.e. removing assets or taking loans). This is done to prevent buyers of accounts from being scammed via frontrunning when selling. The issue is that the cooldown period is only 5 minutes. This opens up the potential for blockstuffing to circumvent the cooldown. 

## Vulnerability Detail

[AccountV1.sol#L265-L270](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L265-L270)

    function transferOwnership(address newOwner) external onlyFactory notDuringAuction {
        if (block.timestamp <= lastActionTimestamp + COOL_DOWN_PERIOD) revert AccountErrors.CoolDownPeriodNotPassed();


        // The Factory will check that the new owner is not address(0).
        owner = newOwner;
    }

When transferring the ownership of an account, it is required that at least 5 minutes has elapsed since the last action. As mentioned above this is designed to prevent frontrunning buys. This protection is potentially not fully adequate due to blockstuffing. 

The main network this is intended to be deployed on is Base, which is an OP chain. L1 submitted transactions are typically included AHEAD of transactions sent directly to the sequencer. As stated in the [docs](https://docs.optimism.io/stack/protocol/outages):

    Sequencers will typically choose to include transactions sent to the OptimismPortal contract before any other transactions but this is not guaranteed

Although not guaranteed, there is still reason for an attacker to pursue this. The worst case scenario is that the sale of the account reverts (they only lose the gas costs) while the best case scenario is they are able to drain the account and the buyer is left with an empty account.

In addition to the attack path present on Base, this is intended to be deployed on other rollups. Rollups like Polygon CDK will utilize a typical fee market like ETH which has public transactions. This would make this tactic extremely effective on such rollups as 5 minutes is a relatively short amount of time to stuff blocks.

## Impact

Drained accounts can be sold to unsuspecting buyers

## Code Snippet

[AccountV1.sol#L265-L270](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/accounts/AccountV1.sol#L265-L270)

## Tool used

Manual Review

## Recommendation

`COOL_DOWN_PERIOD` should be set to a longer value (i.e. 60 minutes) or account locking should be redesigned to further protect buyers.