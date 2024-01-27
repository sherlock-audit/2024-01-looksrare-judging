Great Carmine Fox

medium

# UniswapV3 TWAP oracle is not suitable to get prices on base

## Summary
UniswapV3 TWAP oracle is not suitable to get prices on base.

## Vulnerability Detail
The YOLOv2 readme mentions that the protocol will be deployed on base. The base blockchain is built on the optimism stack and the Uniswap docs mention that UniwsapV3 TWAP oracles should not be used on optimism because [the high-latency block.timestamp update process makes the oracle much less costly to manipulate](https://docs.uniswap.org/concepts/protocol/oracle#oracles-integrations-on-layer-2-rollups).

This is relevant to the protocol because the TWAP prices are used:
 - In [`_deposit()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1163) to determine the amount of entry tickets the depositor should receive in exchange for his tokens
 - In [`_protocolFeeOwedInLOOKS()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1674) to determine the amount of LOOKS necessary to payoff protocol fees

Let's suppose two users, Alice and Bob, already joined the current round in which `valuePerEntry = 0.1ETH` :
- Alice deposited 1ETH and received 10 entry tickets
- Bob deposited 31400LOOKS (=1ETH) and received 10 entry tickets

Charlie wants to join the round as well by depositing 2364 USDC (=1ETH). Before doing that he manipulates the TWAP price of the UniswapV3 USDC/WETH and increases the value of his USDC by 20% (2364 USDC = 1.2ETH), he will now receive 12 entry tickets instead of 10 increasing his chances of winning from 33% (10/30) to 37.5% (12/32).

## Impact

Some users might be able to get an unfair advantage relative to other participants and/or might be able to pay less fees to the protocol.

## Code Snippet
- [`_deposit()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1163)
- [`_protocolFeeOwedInLOOKS()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1674)
## Tool used

Manual Review

## Recommendation

If deploying on base or optimism use a chainlink oracle to retrieve token prices.