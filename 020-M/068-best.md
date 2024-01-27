Nice Brown Griffin

medium

# _shouldDrawWinner overlooking time elapsed condition

## Summary
Due to looksrare docs, there are 3 types of situations mark the end of round:
https://docs.looksrare.org/developers/yolo/yolo-overview#2-how-does-a-round-completes
1.the maximum number of participants is reached
2. the maximum deposits per round is reached（at least two participants）
3. when the time has elapsed and there are at least two participants
But only two situations are taken into consideration in the codebase

## Vulnerability Detail
```solidity
function _shouldDrawWinner(
        uint256 numberOfParticipants,
        uint256 maximumNumberOfParticipants,
        uint256 roundDepositCount
    ) private pure returns (bool shouldDraw) {
        shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
            (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
    }       
```
Should consider time elapsed situation

## Impact
_shouldDrawWinner is used in multiple key functions such as _deposit, rolloverETH, _startRound, depositETHIntoMultipleRounds, if the conditions are missing, it may deviate from the original intention of the project 

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1711-L1719

## Tool used
Manual Review

## Recommendation
Add cutoffTime in the conditional statement