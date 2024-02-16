Docile Mahogany Spider

medium

# The redeeming can be reverted due to VAS in the tranche.

## Summary
`Tranche` is `ERC4626`, and the `Virtual Assets/Shares` are added to prevent inflation attacks. 
However, `redeeming` can be reverted due to this `VAS` even if the `VAS` is small.
## Vulnerability Detail
Junior `tranche` is added to the `pool`. 
The first user deposits `a1` assets, and then he will receive `a1` shares.
```solidity
function convertToSharesAndSync(uint256 assets) public returns (uint256 shares) {
    uint256 supply = totalSupply;  // @audit, totalSupply = 0
    shares = supply == 0 ? assets : assets.mulDivDown(supply + VAS, totalAssetsAndSync() + VAS);
}
```
After that, we assume that a virtual user adds `VAS` virtual assets and gets `VAS` virtual shares.
The second user deposits `a2` assets, and then he will receive `a2` shares.
```solidity
a2 * (a1 + VAS) / (a1 + VAS) = a2
```
Let's suppose a `bad debt` occurs, causing the `liquidity` of this `tranche` to become smaller than `a1 + a2`. (`p < a1 + a2`)

The second user is going to redeem his `shares`. : `w2 = a2 * (p + VAS) / (a1 + a2 + VAS)`
```solidity
function convertToAssetsAndSync(uint256 shares) public returns (uint256 assets) {
    uint256 supply = totalSupply;  // @audit, totalSupply = a1 + a2
    assets = supply == 0 ? shares : shares.mulDivDown(totalAssetsAndSync() + VAS, supply + VAS);
}
```
The first user is going to redeem his `shares`.: `w1 = a1 * (p - w2 + VAS) / (a1 + VAS)`.
If `w1 + w2` is larger than `p`, it means that the first user cannot `redeem` his `shares` because the transaction will be reverted
```solidity
function withdrawFromLendingPool(uint256 assets, address receiver) {
    if (realisedLiquidityOf[msg.sender] < assets) revert LendingPoolErrors.AmountExceedsBalance();  // @audit, p - w2 < w1
}
```
I will solve some mathematical problems to prove this.
```solidity
w1 > p - w2    <=>    a1 * (p - w2 + VAS) / (a1 + VAS) > p - w2   <=>  a1 * (p - w2) + a1 * VAS > (a1 + VAS) * (p - w2)    <=>
a1 * VAS > VAS * (p - w2)   <=>   a1 > p - w2    <=>    a1 > p - (a2 * p + a2 * VAS) / (a1 + a2 + VAS)   <=>
a1 > (p * (a1 + VAS) - a2 * VAS) / (a1 + a2 + VAS)   <=>  a1 * (a1 + a2) + a1 * VAS + a2 * VAS > p * (a1 + VAS)   <=>
(a1 + a2) * (a1 + VAS) > p * (a1 + VAS)    <=>    a1 + a2 > p
```

In other words, the loss also applies to the `virtual user` and this is why the calculated redeeming amount exceeds the actual balance.
## Impact

## Code Snippet
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Tranche.sol#L303-L308
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/Tranche.sol#L328-L333
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/lending-v2/src/LendingPool.sol#L377
## Tool used

Manual Review

## Recommendation
```solidity
function withdrawFromLendingPool(uint256 assets, address receiver) {
-     if (realisedLiquidityOf[msg.sender] < assets) revert LendingPoolErrors.AmountExceedsBalance();
+    if (assets > realisedLiquidityOf[msg.sender]) assets = realisedLiquidityOf[msg.sender];
}
```