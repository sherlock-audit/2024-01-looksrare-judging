Damaged Tangelo Newt

medium

# User cannot withdraw any deposits from a canceled round if they are blacklisted from USDC

## Summary
If a lottery round is cancelled, users can retrieve their deposited assets via `withdrawDeposits()` and get back whatever they deposited. Multiple assets can be deposited for a single round. 

If Alice deposits various assets including USDC into a round, and then gets blacklisted by USDC and the round is cancelled, the call to `withdrawDeposits()` attempts to withdraw all the assets deposited. This will fail because the USDC transfer to Alice will fail. 

## Vulnerability Detail
1. Alice deposits 10 APE, 0.01 ETH and 100 USDC into a round with `_deposit()`. 
2. The round is cancelled for any reason, for example because she was the only player for that round.
3. Alice gets blacklisted by USDC sometime during the round, or anytime after the rounds cancellation but before she withdraws her deposits with `withdrawDeposits()`.
4. Alice calls `withdrawDeposits()` for the round she participated in. This loops through the specific round and gets each asset she deposited and attempts to transfer it back to her through `_transferTokenOut()` and `_executeERC20DirectTransfer()`. 
5. The `_transferTokenOut()` attempts to send ETH, APE, and USDC. It reverts when it attempts to send USDC because she is blacklisted, and the entire transaction fails.
6. There is no mechanism to withdraw only certain assets deposited from a specific round. If all tokens deposited during a single round cannot be withdrawn all at once, then the `withdrawDeposits()` function reverts and none of the assets can be withdrawn.

## Impact
User cannot withdraw any of the other tokens they deposited if they become blacklisted by USDC.

## Code Snippet
`_transferTokenOut()`: https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1483
`withdrawDeposits()`: https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L625
`withdrawDeposits()`: https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L630

## Tool used
Manual Review

## Recommendation
* Consider not including tokens with blacklists.
* Consider adding a function that can allow users to withdraw their deposits a single token at a time.