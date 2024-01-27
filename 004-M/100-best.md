Jumpy Burlap Mockingbird

medium

# Inconsistency in Pause Mechanism

## Summary
`_startRound` function has some inconsistent behavior with respect to the paused state. 
While the pause mechanism effectively prevents direct deposits and round drawing actions, it does not stop the initiation of new rounds and as long as users can deposit into multiple rounds, this paused mechanisms preventing deposit is broken and does not enforce fairness

## Vulnerability Detail
The `_startRound` function can be invoked in various scenarios : after a round is cancelled, when a round is drawn, or during contract initialization. 

However, the function does not comprehensively account for the paused state of the contract in its logic :

```solidity
function _startRound(uint256 _roundsCount) private returns (uint256 roundId) {
    ...
    if ( !paused() &&  _shouldDrawWinner(numberOfParticipants, round.maximumNumberOfParticipants, round.deposits.length)) {
        _drawWinner(round, roundId);
    } else {
        uint256 roundSlot = _getRoundSlot(roundId);
        assembly {
            let roundValue := or(sload(roundSlot), 1)
            if gt(numberOfParticipants, 0) {
                roundValue := or(roundValue, shl(ROUND__CUTOFF_TIME_OFFSET, add(timestamp(), _roundDuration)))
            }
            sstore(roundSlot, roundValue)
        }
        emit RoundStatusUpdated(roundId, RoundStatus.Open);
    }
    ...
}
```
If paused is true, users cannot deposit using standard functions. However, those who have previously used `depositETHIntoMultipleRounds()` can still have their deposits counted in the newly started round. 
This creates an imbalance for two scenarios ,suppose some users used `depositETHIntoMultipleRounds` : 
1. and the round `cutoffTime` has passed, a winner should be chosen to win the round, but `drawWinner()` function has `whenNotPaused` modifier implemented which prevents a winner to be randomly chosen, however `cancel()` hasn't `whenNotPaused` modifier which allows any external user to cancel a round with enough depositors and deposits in it and where a winner should win.
2. If `paused` is toggled to false during an ongoing round, it could unfairly shorten the window for new participants to join, benefiting those who already deposited.

## Impact
1. Gameplay Fairness: The inconsistent handling of the pause state can lead to situations where certain players are favored over others, breaking the fairness of the game.

2. Strategic Disruption: Players might exploit the pause mechanism to gain an advantage, leading to strategic disruptions and potential manipulation of the game's outcome.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L979

## Tool used

Manual Review

## Recommendation

Review the paused mechamism implementation , If paused is true in `_startRound` it should pause the contract until it is toggled 