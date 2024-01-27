Great Carmine Fox

medium

# Solidity version `0.8.23` might not be compatible with some chains

## Summary
Solidity version `0.8.23` might not be compatible with some chains.

## Vulnerability Detail

The contest readme mentions that the protocol will be deployed on Arbitrum but the [solidity version `0.8.23`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L2) used in the `YOLOv2.sol` contract is [incompatible with Arbitrum](https://docs.arbitrum.io/for-devs/concepts/differences-between-arbitrum-ethereum/solidity-support) due to the introduction of the [`PUSH0` opcode in solidity versions >= `0.8.20`](https://github.com/ethereum/solidity/releases/tag/v0.8.20).

## Impact

In the best-case scenario the deployment on a chain that does not support the `PUSH0` opcode just reverts, in the worst-case scenario it might produce unpredictable outcomes.

## Code Snippet
- [pragma solidity 0.8.23](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L2)
## Tool used

Manual Review

## Recommendation

Change the solidity version to `0.8.19`.