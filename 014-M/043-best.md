Mythical Satin Tuna

high

# Rounds can not be immediately drawn after fulfillRandomWords due to VRF contracts reentrancy guard

## Summary
Users can deposit ETH to multiple future rounds. If the future round can be "drawn" immediately when the current round ends this execution will revert because of chainlink contracts reentrancy guard protection hence ending the current round will be impossible and it needs to be cancelled by governance.
## Vulnerability Detail
When the round is Drawn, VRF is expected to call `fulfillRandomWords` to determine the winner in the current round and starts the next round.
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1293

`fulFillRandomWords` function is triggered by the `rawFulfillRandomWords` external function in the Yolo contract which is also triggered by the randomness providers call to VRF's `fulFillRandomWords` function which has a reentrancy check as seen and just before the call to Yolo contract it sets to "true".
https://github.com/smartcontractkit/chainlink/blob/6133df8a2a8b527155a8a822d2924d5ca4bfd122/contracts/src/v0.8/vrf/VRFCoordinatorV2.sol#L526-L546


If the next round is "drawable" then the `_startRound` will [draw the next round immediately](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L964-L967) and by doing so it will try to call VRF's `requestRandomWords` function which also has a nonreentrant modifier
https://github.com/smartcontractkit/chainlink/blob/e4bde648582d55806ab7e0f8d4ea05a721ca120d/contracts/src/v0.8/vrf/VRFCoordinatorV2.sol#L349-L355

since the reentrancy lock was set to "true" in first call, the second calling requesting randomness will revert. Current "drawn" round will not be completed and there will be no winner because of the chainlink call will revert every time the VRF calls the yolo contract. 

## Impact
Already "drawn" round will not be concluded and there will be no winner.
VRF's keepers will see their tx is reverted
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L949-L991

https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1270-L1296

https://github.com/smartcontractkit/chainlink/blob/e4bde648582d55806ab7e0f8d4ea05a721ca120d/contracts/src/v0.8/vrf/VRFCoordinatorV2.sol#L349-L408

https://github.com/smartcontractkit/chainlink/blob/e4bde648582d55806ab7e0f8d4ea05a721ca120d/contracts/src/v0.8/vrf/VRFCoordinatorV2.sol#L526-L572
## Tool used

Manual Review

## Recommendation
Do not draw the contract immediately if it's winnable. Anyone can call drawWinner when the conditions are satisfied. This will slow the game pace, if that's not ideal then add an another if check to the drawWinner to conclude a round if its drawable immediately .