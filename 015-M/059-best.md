Zealous Alabaster Fly

high

# `CREATE2` address collision against an Account will allow complete draining of lending pools

## Summary

The factory function `createAccount()` creates a new account contract for the user using `CREATE2`. We show that a meet-in-the-middle attack at finding an address collision against an undeployed account is possible. Furthermore, such an attack allows draining of all funds from the lending pool.

## Vulnerability Detail

The attack consists of two parts: Finding a collision, and actually draining the lending pool. We describe both here:

### PoC: Finding a collision

Note that in `createAccount`, `CREATE2` salt is user-supplied, and `tx.origin` is technically also user-supplied:

```solidity
account = address(
    new Proxy{ salt: keccak256(abi.encodePacked(salt, tx.origin)) }(
        versionInformation[accountVersion].implementation
    )
);
```

The address collision an attacker will need to find are:
- One undeployed Arcadia account address (1).
- Arbitrary attacker-controlled wallet contract (2).

Both sets of addresses can be brute-force searched because:
- As shown above, `salt` is a user-supplied parameter. By brute-forcing many `salt` values, we have obtained many different (undeployed) wallet accounts for (1).
- (2) can be searched the same way. The contract just has to be deployed using `CREATE2`, and the salt is in the attacker's control by definition.

An attacker can find any single address collision between (1) and (2) with high probability of success using the following meet-in-the-middle technique, a classic brute-force-based attack in cryptography:
- Brute-force a sufficient number of values of salt ($2^{80}$), pre-compute the resulting account addresses, and efficiently store them e.g. in a Bloom filter data structure.
- Brute-force contract pre-computation to find a collision with any address within the stored set in step 1.

The feasibility, as well as detailed technique and hardware requirements of finding a collision, are sufficiently described in multiple references: 
- [1](https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90): A past issue on Sherlock describing this attack.
- [2](https://eips.ethereum.org/EIPS/eip-3607): EIP-3607, which rationale is this exact attack. The EIP is in final state.
- [3](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses): A blog post discussing the cost (money and time) of this exact attack.

The [hashrate of the BTC network](https://www.blockchain.com/explorer/charts/hash-rate) has reached $6 \times 10^{20}$ hashes per second as of time of writing, taking only just $33$ minutes to achieve $2^{80}$ hashes. A fraction of this computing power will still easily find a collision in a reasonably short timeline. 

### PoC: Draining the lending pool

Even given EIP-3607 which disables an EOA if a contract is already deployed on top, we show that it's still possible to drain the lending pool entirely given a contract collision.

Assuming the attacker has already found an address collision against an undeployed account, let's say `0xCOLLIDED`. The steps for complete draining of a lending pool are as follow:

First tx:
- Deploy the attack contract onto address `0xCOLLIDED`.
- Set infinite allowance for {`0xCOLLIDED` ---> attacker wallet} for any token they want.
- Destroy the contract using `selfdestruct`.
    - Post Dencun hardfork, [`selfdestruct` is still possible if the contract was created in the same transaction](https://eips.ethereum.org/EIPS/eip-6780). The only catch is that all 3 of these steps must be done in one tx.

The attacker now has complete control of any funds sent to `0xCOLLIDED`. 

Second tx:
- Deploy an account to `0xCOLLIDED`. 
- Deposit an asset, collateralize it, then drain the collateral using the allowance set in tx1.
- Repeat step 2 for as long as they need to (i.e. collateralize the same asset multiple times).
    - The account at `0xCOLLIDED` is now infinitely collateralized.
    - Funds for step 2 and 3 can be obtained through external flash loan. Simply return the funds when this step is finished.
- An infinitely collateralized account has infinite borrow power. Simply borrow all the funds from the lending pool and run away with it, leaving an infinity collateral account that actually holds no funds.

The attacker has stolen all funds from the lending pool.

### Coded unit-PoC

While we cannot provide an actual hash collision due to infrastructural constraints, we are able to provide a coded PoC to prove the following two properties of the EVM that would enable this attack:
- A contract can be deployed on top of an address that already had a contract before.
- By deploying a contract and self-destruct in the same tx, we are able to set allowance for an address that has no bytecode.

Here is the PoC, as well as detailed steps to recreate it:
1. Paste the following file onto Remix (or a developing environment of choice): https://gist.github.com/midori-fuse/087aa3248da114a0712757348fcce814
2. Deploy the contract `Test`.
3. Run the function `Test.test()` with a salt of your choice, and record the returned address. The result will be:
    - `Test.getAllowance()` for that address will return exactly `APPROVE_AMOUNT`.
    - `Test.getCodeSize()` for that address will return exactly zero.
    - This proves the second property.
4. Using the same salt in step 3, run `Test.test()` again. The tx will go through, and the result will be:
    - `Test.test()` returns the same address as with the first run.
    - `Test.getAllowance()` for that address will return twice of `APPROVE_AMOUNT`.
    - `Test.getCodeSize()` for that address will still return zero.
    - This proves the first property.

The provided PoC has been tested on Remix IDE, on the Remix VM - Mainnet fork environment, as well as testing locally on the Holesky testnet fork, which as of time of writing, has been upgraded with the Dencun hardfork.

## Impact

Complete draining of a lending pool if an address collision is found.

With the advancement of computing hardware, the cost of an attack has been shown to be [just a few million dollars](https://mystenlabs.com/blog/ambush-attacks-on-160bit-objectids-addresses), and that the current Bitcoin network hashrate allows about $2^{80}$ in about half an hour. The cost of the attack may be offsetted with longer brute force time.

For a DeFi lending pool, it is normal for a pool TVL to reach tens or hundreds of millions in USD value (top protocols' TVL are well above the billions). It is then easy to show that such an attack is massively profitable.

## Code Snippet

https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/Factory.sol#L96-L100

## Tool used

Manual Review, Remix IDE

## Recommendation

The mitigation method is to prevent controlling over the deployed account address (or at least severely limit that). Some techniques may be:
- Do not allow a user-supplied `salt`, as well as do not use the user address as a determining factor for the salt.
- Use the vanilla contract creation with `CREATE`, as opposed to `CREATE2`. The contract's address is determined by `msg.sender` (the factory), and the internal `nonce` of the factory (for a contract, this is just "how many other contracts it has deployed" plus one).

This will prevent brute-forcing of one side of the collision, disabling the $O(2^{81})$ search technique.
