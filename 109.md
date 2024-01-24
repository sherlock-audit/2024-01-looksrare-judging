Jumpy Butter Pig

high

# Deployment in `Linea`  blockchain will not be possible

## Summary
The protocol is designed to be deployed on potentially any EVM-compatible Layer 2 (including ZK L2s). However, assuming deployment on the Linea mainnet, the deployment may not be possible due to the compiler.
## Vulnerability Detail
The protocol is currently utilizing Compiler version `0.8.23`, and there is an expectation to deploy it on `Linea`. However, it's important to note that Linea does not support compiler version `0.8.23`. refer to: https://docs.linea.build/build-on-linea/ethereum-differences. 
## Impact
Deployment will fail on Linea blockchain
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L2
## Tool used

Manual Review

## Recommendation
Use Solidity version 0.8.19 or lower and EVM version to `London` for Linea mainnet.