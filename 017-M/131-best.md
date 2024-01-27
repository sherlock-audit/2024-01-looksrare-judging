Short Chiffon Bird

medium

# users can't specify round ID in depositETHIntoMultipleRounds and rollover function their deposits may put into different rounds and cause them loss

## Summary
when users deposits token with `depositETHIntoMultipleRounds()`, code deposits users token to current round. the issue is that user may want to deposit to current round but their tx stay in mempool for while and when their tx executed the current round has been changed and their funds get deposited into the wrong round. This is like slippage protection in the AMMs, the current behavior doesn't protect users from this and users may participate in a random round while they wanted to explicitly get into the specific round.

this issue exists for rollover function too.

## Vulnerability Detail
This is `depositETHIntoMultipleRounds()` code, as you can see function doesn't have roundId and it start to deposit user tokens for current round `roundsCount` and users can't specify the round they want to deposit into:
```javascript
    function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {
        uint256 numberOfRounds = amounts.length;
        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

        uint256 startingRoundId = roundsCount;
        Round storage startingRound = rounds[startingRoundId];
        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);
```

This makes users to become vulnerable for slippage scenarios. for example users see a good opportunity to deposit into the current round(ID1) and they want only to deposit into this ID1 round and they sign their transaction, but their transaction stuck in the mempool and when it's mined the current round is changed and it's not ID1 and user may lose their funds as this round is not according to their strategy.

it's true that the winner is random and it looks like there is no different between rounds but for these reasons the rounds can be different:
1. different round mean different price for ERC20 and ERC721 tokens and user may want to deposit his ETH into a round that have lower price for ERC20 or ERC721.
2. different rounds may have different parameters like duration or `valuePerEntry`.
3. user may have strategy to deposit to specific rounds.

also attacker can use front-run and fill the current rounds and cause user funds to be deposited into specific round that is not favorable for user. for example depositing huge ETH into a round that has low entry amount will result in loss(if a user deposit 100ETH and another user deposit 1ETH and the FEE is 10% then even so user1 will have higher chance of winning but the fee will be bigger than extra winning amount)

## Impact
users funds will be deposited into random rounds that user didn't intended to do it. users can't perform their strategy and they may lose funds in the rounds that they didn't want to participate. 

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L312-L321

## Tool used
Manual Review

## Recommendation
function should get roundID from the user and deposit ETH into those rounds.