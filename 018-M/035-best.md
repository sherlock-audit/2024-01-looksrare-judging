Mythical Satin Tuna

medium

# WETH can't be used as payment token

## Summary
WETH token can't be used as payment token because the pricing of TWAP always respects the -WETH pairs and since there are no WETH-WETH pools getting a WETH price from any Univ3 pool will be impossible. 
## Vulnerability Detail
The price of every token is obtained from a Uniswap V3 pool, and it is relative to ETH, so the pool has to be a token-WETH pair.
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L54-L66

Since there will be no pool as a WETH-WETH pair to get WETH's price, acquiring WETH's price will be impossible from any Uni pool with the current PriceOracle.

Hence, the WETH token will not be possibly used as a payment token, although the Sherlock doc says that every ERC20 token with decent liquidity is a candidate for a payment token.
## Impact
Contest sherlock doc states that every ERC20 with decent liquidity can be used as payment token. WETH is probably the most liquid token in the crypto space and it can't be used as payment token. Probably, instead of WETH, ETH is used which are the same thing, but since contest's Sherlock doc says that "every ERC20 with decent liquidity" I will submit this issue as a medium. 
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/PriceOracle.sol#L54-L66
## Tool used

Manual Review

## Recommendation