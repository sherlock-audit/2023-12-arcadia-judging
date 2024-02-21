Bent Misty Sardine

high

# Incorrect Implementation Of Virtual Shares For ERC4626 Inflation Mitigation

## Summary

The same variable (`VAS`) is used as both the virtual asset and virtual shares which differs from OpenZeppelin's Virtual Shares Implementation. The difference in Arcadia's implementation makes it still vulnerable to the inflation attack.

## Vulnerability Detail

In Arcadia The same variable (VAS) is used as both the virtual asset and virtual shares.Additionally, the shares minted are the same as assets on the initial deposit. This differs from the OpenZeppelin Virtual Shares Implementation, which adds 1 to asset and a large number like 1e3 to shares. In OZ's version, the virtual share logic is also applied to the initial deposit rather than using seperate logic for it.

In OpenZeppelin's implementation, even on the first deposit it is impossible to mint less than the initial share amount. For example depositing 1 asset gives 1e3 shares. Note that this share amount is maintained no matter how many assets are donated to the pool. 

In contrast, in Arcadia, the shares for the initial deposit is equal to the share. Therefore, if the first deposit is 1 asset then they get 1 share. This low share amount allows for the inflation attack.

Aracadia's version:

```solidity
    function convertToShares(uint256 assets) public view override returns (uint256 shares) {
        // Cache totalSupply.
        uint256 supply = totalSupply;


        shares = supply == 0 ? assets : assets.mulDivDown(supply + VAS, totalAssets() + VAS);
    }
```

OpenZeppelin's Version:

```solidity
    function _convertToShares(uint256 assets, Math.Rounding rounding) internal view virtual returns (uint256) {
        return assets.mulDiv(totalSupply() + 10 ** _decimalsOffset(), totalAssets() + 1, rounding);
    }
```

Lets say that VAS is 1e3

1. deposit 1 asset to get 1 share
2. donate 1e18 assets to the pool
3. Now each subsequent deposit or withdrawal has a rounding error of 1e18.

We can break down the impact of the "share inflation attack into 2 impacts":

1. first depositor loses their initial deposit
2. all subsequent deposits, even after deposits get a rounding error equivalent to the initial donation

With this simple sequence we have achieved impact #2. If $100 USD of assets is donated, then 1 share is worth $100, and therefore each withdrawal and deposit which may cause 1 share of rounding loss can lose up to $100 worth of funds.

This attack does not work with the OpenZeppelin impementation of +1, +1e3, because you can never mint just 1 share. With OpenZeppelin's implementation, to cause the same rounding loss you would need to deposit $100K to cause a rounding loss of $100 for each share.


Achieving impact #1 requires bypassing another anti-inflation attack mechanism. The traditional inflation attack requires rounding down the victim's share to 0. Basically, the frontrunning donation is slightly higher than the deposit amount, which makes the donation round from ~0.99 shares (if rounding loss wasn't a thing) to 0 shares.

There is a safeguard which makes the trasnaction revert from this case.

The orignial attack  _"make victim round from `0.999999` to `0` to steal all their deposit"_
 
The bypass for the 0 share check is _"make victim round from `1.99999` to `1` to steal a chunk of their deposit"_

Let's show an example here. We have already established that we can manipulate the initial share and asset values, so we will just set up the state that happens before the victim deposits. Tranche state:

```solidity
VAS = 1e3 

totalShares = 1e5 

totalAssets = 0.5001e10
```

The equation from Arcadia is 

```solidity
shares = assets.mulDivDown(supply + VAS, totalAssets() + VAS);
```

```solidity
shares = 1e5 * (1e5 +1e3 ) / (0.51e10  + 1e3)
=  10100000000 / 5100001000
= 1.98
//round down!
= 1
```

The victims gets basically half the shares they are "supposed" to due to the rounding error.

The technique for bypassing the 0 share check is also descibed in [this repo](https://github.com/ZeframLou/bunni). Scroll down in the bottom of the linked page to _"Frontrunning the first deposit may steal 1/4 of the deposit"_

## Impact

Immediate loss of funds for the first depositor, and large rounding loss for all subsequent depositors. 

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Tranche.sol#L290-L295

## Tool used

Manual Review

## Recommendation

Implement OpenZeppelin's Virtual Share mitigation. This uses 1 asset and a large number of shares.