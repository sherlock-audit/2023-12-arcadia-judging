Hidden Concrete Lemur

high

# Insufficient access control allows any user to whitelist a pool which contradicts the intended functionality

## Summary

## Vulnerability Detail
The StarGate Module implements an addAsset() which  functions  as a means to add StarGate  Pools to the registry and marks the pool as an allowed asset.
Adding a pool in other AssetModules have access restriction(i.e Only the Owner of the AssetModule can add a pool)
for example (ERC20TokenPrimaries) and according to the natspec it clearly indicates i quote " dev No end-user should directly interact with the Stargate Asset Module, only the Registry, the contract owner or via the actionHandler".The issue is that the addAsset lacks access control which would allow any user  to add any starGate pool only if the underlying asset is allowed in the registry which clearly contradicts the docs. 

## Impact: None i guess

## Code Snippet:
https://github.com/sherlock-audit/2023-12-arcadia/blob/main/accounts-v2/src/asset-modules/Stargate-Finance/StargateAM.sol#L62


## Tool used

Manual Review

## Recommendation:Add Access control on the addAsset() 
