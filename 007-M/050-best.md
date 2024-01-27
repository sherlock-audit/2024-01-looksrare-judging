Mythical Satin Tuna

high

# Rounds first ERC20 depositor may have fewer entries than they have previewed

## Summary
Users can deposit whitelisted ERC20 tokens to participate in a round. The pricing of ERC20 tokens is determined via TWAP and quoted in ETH terms. Due to TWAP lags and the uncertainty of transaction mining timing, users may end up with fewer entry tickets than intended when sending the transaction.
## Vulnerability Detail
When users deposit ERC20 to participate in the round, this is how the entry count of the user is calculated:
```solidity
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
                    if (entriesCount == 0) {
                        revert InvalidValue();
                    }
```
The very first depositor of the ERC20 token in a round determines the price for the others. 
Also as we can see the entry count is round down by deposits. Since the tx sent to network is not guaranteed to be executed in the current block that the user queried the TWAP price, user can get lesser entries than supposed to. 

**Textual PoC:**
Suppose the round's valuePerEntry is 1 ether, and a user (Alice) is depositing USDC to participate in the round. Assume that 1 ETH is valued at $2000 USDC in the TWAP before Alice sends the transaction to the network. Alice aims to get exactly 2 entries, requiring her to deposit precisely 4000 USDC. If she deposits 3999 USDC, she will receive only 1 entry; if she deposits 4001 USDC, she will get 2 entries. Therefore, there is no reason for Alice to deposit a lesser or greater amount than she should because of the rounding.

Since Alice cannot guarantee that the transaction she sent to the network will be mined in the current block where she viewed the TWAP price, and she has no idea whether the next block's TWAP price will change slightly, she can easily end up with 1 entry instead of 2. Here are a couple of scenarios demonstrating how this can happen:

Alice sends 4000 USDC, which is the current TWAP price at the time she viewed it in the frontend of the app at block number "x." Alice's transaction is mined at block "x+2," and between those 2 blocks, the price changes slightly. Now, TWAP indicates the price of 1 ETH is 2000.0230 USDC, resulting in Alice receiving only 1 entry when she intended to get 2 entries with exactly 4000 USDC. This means Alice deposited an extra 1999.977 USDC that didn't contribute to her entry count.

Alice sends 4000 USDC, which is the current TWAP price at the time she viewed it in the frontend of the app at block number "x." TWAP prices are updated only per block. Alice's transaction can be executed at best at block "x+1," the very next block to be mined. Just before Alice's transaction, there is a swap executed in the pool that TWAP queried, and the price now changed to 1 ETH = 2000.0230 USDC. Alice again buys only 1 entry and spends 1999.977 USDC for nothing.
## Impact
As users are not encouraged to deposit slightly more or less than the exact TWAP-based price for ERC20 deposits, there is a risk of various scenarios outlined in the detailed section. These scenarios involve changes in the TWAP price, leading users to deposit unnecessary tokens and receive fewer entries than anticipated. In such cases, funds are spent without obtaining an entry, hence, I'll label this as high.
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1161-L1178
## Tool used

Manual Review

## Recommendation
Introduce a variable named "minimumEntries" to "Deposits" struct and ensure that the user depositing ERC20 tokens has to get the minimum entry deposit count on every deposit or revert.