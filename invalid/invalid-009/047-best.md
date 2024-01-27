Great Carmine Fox

medium

# A round can be canceled after a random number has been drawn in a specific edge case

## Summary
There's an edge case in which the protocol fairness can be undermined by canceling a round after a random number has been drawn.

## Vulnerability Detail
The [`cancelAfterRandomnessRequest()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L459) function allows anybody to cancel the current round if the round status is `Drawn` and at least 1 day (24 hours) passed since the randomness request.


To understand the root cause of this issue there's two things to keep in mind:
1. A round can be cancelled if least 1 day (24 hours) passed since the randomness request
2. If the LINK balance of the chainlink VRF subscription is below the minimum required a request for randomness will remain pending for up to 24 hours

This might look fine in theory but in practice there's a small time window in which an user can cancel a round if the random number sent by chainlink is not in their favor. 

### POC
Here's how it works:
1. A request for randomness is sent to chainlink VRF via [`_drawWinner()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1001-L1007)
2. The LINK balance of the subscription is below the minimum required to fulfill the request
3. Alice who is participating in the round notices this and funds the chainlink subscription with LINKs right before the 24 hours mark
4. The chainlink node sends the randomness response to YOLOV2 but the transaction stays pending in the mempool for some minutes
5. Alice monitors the mempool and checks if the random number sent by chainlink is in her favor (ex. she's the winner) or not
6. If she's not the winner she can fronturn the randomness response and cancel the round, she can do this because the 24 hours mark is passed from the point of view of [`cancelAfterRandomnessRequest()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L459)

Important to note is that the delay between `4.` and `.6` can happen because:
- The attackers times everything perfectly
- The network is congested
- The attacker [fills blocks on purpose](https://medium.com/hackernoon/the-anatomy-of-a-block-stuffing-attack-a488698732ae) to delay the chainlink response

## Impact
Although this requires some preconditions and is unlikely to happen, there's a possibility of undermining the fairness of the protocol which is one of his [main selling points](https://docs.looksrare.org/developers/yolo/yolo-overview):
> The YOLO system is designed to operate a transparent, fair, and decentralized game directly on the Ethereum blockchain.

## Code Snippet
- [`cancelAfterRandomnessRequest()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L459)
## Tool used

Manual Review

## Recommendation

Increase the time window after which [`cancelAfterRandomnessRequest()`](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L451-L468) can be called by at least 10 minutes on mainnet and potentially more on cheaper chains:
```solidity
if (block.timestamp < round.drawnAt + 1 days + 10*60) {
    revert DrawExpirationTimeNotReached();
}
```