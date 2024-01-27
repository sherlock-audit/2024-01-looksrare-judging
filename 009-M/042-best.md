Acrobatic Flint Lobster

medium

# Failed Deployment

## Summary

The hardcoded deployment address for [`UNISWAP_V3_FACTORY`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18) is not compatible with all intended deployment targets.

## Vulnerability Detail

The [`PriceOracle`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18) has hardcoded the address of the [`UNISWAP_V3_FACTORY`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18), which is valid for **Mainnet**, **Goerli**, **Arbitrum**, **Optimism** and **Polygon**, and shares the following deployment addresses (`0x1F98431c8aD98523631AE4a59f267346ea31F984`):

[Etherscan (Mainnet)](https://etherscan.io/address/0x1F98431c8aD98523631AE4a59f267346ea31F984#code)
[Etherscan (Goerli)](https://goerli.etherscan.io/address/0x1F98431c8aD98523631AE4a59f267346ea31F984#code)
[Etherscan (Optimism Mainnet)](https://optimistic.etherscan.io/address/0x1F98431c8aD98523631AE4a59f267346ea31F984#code)
[Arbiscan](https://arbiscan.io/address/0x1F98431c8aD98523631AE4a59f267346ea31F984#code)
[Polygonscan](https://polygonscan.com/address/0x1F98431c8aD98523631AE4a59f267346ea31F984#code)

However, as stated in the contest `README`, YOLOv2 is also additionally intended to be deployed on **Base**, where the [`IUniswapV3Factory`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18) is deployed to a separate CREATE2 address, `0x33128a8fC17869897dcE68Ed026d694621f6FDfD`:

[Basescan](https://basescan.org/address/0x33128a8fC17869897dcE68Ed026d694621f6FDfD#code)

## Impact

Breaks core contract functionality on Base (Medium).

## Code Snippet

```solidity
IUniswapV3Factory private constant UNISWAP_V3_FACTORY =
        IUniswapV3Factory(0x1F98431c8aD98523631AE4a59f267346ea31F984);
```

## Tool used

Manual Review

## Recommendation

Elevate [`UNISWAP_V3_FACTORY`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18) to an `immutable` variable initialized upon construction of a [`PriceOracle`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L18).

