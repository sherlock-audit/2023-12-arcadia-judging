Orbiting Pineapple Python

medium

# Dilution of Donations in Tranche

## Summary
In this attack, the attacker takes advantage of the non-atomic nature of the donation and the share valuation process. By strategically placing deposit and withdrawal transactions around the donation transaction, the attacker can temporarily inflate their share of the pool to capture a large portion of the donated funds, which they then quickly exit with, leaving the pool with their original investment plus extra value extracted from the donation.

## Vulnerability Detail
Though there is no reasonable flow where users will just 'donate' assets to others, Risk Manager may needs to call `donateToTranche` to compensate the jrTranche after an auction didn't get sold and was manually liquidated after cutoff time or in case of bad debt. 

`donateToTranche` function of a lending pool smart contract, allows for a sandwich attack that can be exploited by a malicious actor to dilute the impact of donations made to a specific tranche. This attack involves front-running a detected donation transaction with a large deposit and following it up with an immediate withdrawal after the donation is processed.

The lending pool contract in question allows liquidity providers (LPs) to deposit funds into tranches, which represent slices of the pool's capital with varying risk profiles. The `donateToTranche` function permits external parties to donate assets to a tranche, thereby increasing the value of the tranche's shares and benefiting all LPs proportionally. Transactions can be observed by one of the LP's before they are mined. An attacker can exploit this by identifying a pending donation transaction and executing a sandwich attack. This attack results in the dilution of the donation's intended effect, as the attacker's actions siphon off a portion of the donated funds.

## Impact
Dilution of Donation: The intended impact of the donation on the original LPs is diluted as the attacker siphons off a portion of the donated funds.

## Steps to Reproduce Issue
1. Front-Running: The attacker deposits a significant amount of assets into the target tranche before the donation transaction is confirmed, temporarily increasing their share of the tranche.

2. Donation Processing: The original donation transaction is processed, increasing the value of the tranche's shares, including those recently acquired by the attacker.

3. Back-Running: The attacker immediately withdraws their total balance from the tranche, which now includes a portion of the donated assets, effectively extracting value from the donation meant for the original LPs.

## Code Snippet
 [Code Snippet](https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L350)

## Coded PoC
```solidity
https://github.com/Atharv181/Arcadia-POC
```
```javascript
- git clone https://github.com/Atharv181/Arcadia-POC
- cd Arcadia-POC
- forge install
- forge test --mt test_PoC -vvv
```

## Tool used

Manual Review, Foundry

## Recommendation
- Snapshot Mechanism: Take snapshots of share ownership at random intervals and distribute donations based on the snapshot to prevent exploitation.
- Timelocks: Implement a timelock mechanism that requires funds to be locked for a certain period before they can be withdrawn.

