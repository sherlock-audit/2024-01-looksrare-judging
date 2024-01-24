Short Chiffon Bird

high

# attacker can deposit ERC20 and ERC721 tokens with wrong type and receive huge amount of entries

## Summary
users can deposit whitelisted ERC20 and ERC721 tokens and receive entries based on deposited tokens value. the issue is that there is only one whitelist token for ERC20 and ERC721 and code trust the token type user defined and user can define ERC20 type for actual NFT token or vice versa. this would cause wrong price and entry calculation and user would receive tremendous amount of entry. of course attacker need to create a Uniswap pool for NFT token or Reservoir oracle to support ERC20 tokens.

## Vulnerability Detail
This is part of `_deposit()` code:
```javascript
            for (uint256 i; i < deposits.length; ++i) {
                DepositCalldata calldata singleDeposit = deposits[i];
                address tokenAddress = singleDeposit.tokenAddress;
                if (isCurrencyAllowed[tokenAddress] != 1) {
                    revert InvalidCollection();
                }
                uint256 price = prices[tokenAddress][roundId];
                if (singleDeposit.tokenType == YoloV2__TokenType.ERC721) {
               if (price == 0) {
                        price = _getReservoirPrice(singleDeposit);
                        prices[tokenAddress][roundId] = price;
                    }

                    uint256 entriesCount = price / round.valuePerEntry;
                ...................
                ..................
                } else if (singleDeposit.tokenType == YoloV2__TokenType.ERC20) {
                    if (price == 0) {
                        price = erc20Oracle.getTWAP(tokenAddress, uint32(TWAP_DURATION));
                        prices[tokenAddress][roundId] = price;
                    }

                    uint256[] memory amounts = singleDeposit.tokenIdsOrAmounts;
                    if (amounts.length != 1) {
                        revert InvalidLength();
                    }

                    uint256 amount = amounts[0];

                    uint256 entriesCount = ((price * amount) / (10 ** IERC20(tokenAddress).decimals())) /
                        round.valuePerEntry;

```
as you can see code uses the input value `deposits[]` to check the token type and trust the user input. after that code calculates tokens value based on the type of the token. attacker can use this and fool the system and set a ERC20 token as ERC721 or vice versa. this would result in wrong value calculation and wrong entries count. attacker can perform one of these two:
1. set ERC20 as ERC721, then if attacker deposits 10 wei tokens code would use oracle to get price of the token and it would return the price 10^18 tokens in ETH(PRICE1) but code would assume that that the price of the 1 NFT token and the total value user would gain would be `10*PRICE1`.
2. set ERC721 as ERC20, then if attacker deposits a NFT token with `ID=(1000 * 1e18)` code would try to get price of the token from uniswap pool. the price is ETH amount for  10^18 token (PRICE1) and as result would calculate the whole deposits value as `1000 * PRICE1`.

because code would use `transferFrom(address, address, uint256)` to transfer ERC20 and ERC721 so there won't be a problem in transferring tokens. the only problem is the oracle price, attacker can bypass it with:

exploiting #1 is possible if Reservoir oracle add support ERC20 tokens price . it's a logical expectation for Reservoir oracle to add a price for ERC20 tokens too in Reservior as they expand their products. As code only checks the token address and signature and timestamp the ERC20 signed by Reservoir would pass the code checks. it would be the price of the 1e18 (18 as decimal) tokens but code would assume it's a NFT and its the price of each NFT.

exploiting #2 is possible if attacker creates a Uniswap pool for the ERC721 token. it's possible to create pool for ERC721 token as pool's code would use `transferFrom(address, address, amount)` to transfer tokens and so attacker can deposit NFTs and ETH as liquidity to the target pool and create a valid price for the NFT token in uniswap pool. 


## Impact
attacker can set wrong token type and as result the calculations will be wrong and attacker would receive a lot of entry point and win the round with very high probability and steal others deposits.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1095-L1110
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1161-L1178

## Tool used
Manual Review

## Recommendation
code shouldn't trust the user input and there should be two different whitelist for ERC20 and ERC721 tokens. 