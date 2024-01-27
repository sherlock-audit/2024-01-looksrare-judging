Dapper Metal Eel

medium

# No Access control on `YoloV2::cancelAfterRandomnessRequest()` as a result anyone can call that cancel function

## Summary
When you read the natspec of the function `cancelAfterRandomnessRequest()` in `IYoloV2` contract, you will realise the mentioned contract is supposed to be executed by the owner but there is no access control making sure that is possible.
## Vulnerability Detail
According to the natspec `Only callable by contract owner` this function is supposed to be called by only the owner, but access controls were not put in place to ensure that so as a result an attacker can call this contract too.

## Impact
Anyone can cancel a round after randomness request if the randomness request does not arrive after a certain amount of time, as a result attackers can take advantage of this function.

## Code Snippet

This is the vulnerable function
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L451-L468


The inherited function from `IYoloV2.sol` which indicates the vulnerability from the natspec
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/interfaces/IYoloV2.sol#L263-L267

```solidity
 /**
            * @notice Cancels a round after randomness request if the randomness request
             *         does not arrive after a certain amount of time.
@-->     *         Only callable by contract owner.
     */
    function cancelAfterRandomnessRequest() external;
```

## Tool used

Manual Review

## Recommendation

This function has to add `_validateIsOwner()` to permit only the owner to access that contract. like this;
```diff
function cancelAfterRandomnessRequest() external nonReentrant {
+        _validateIsOwner();
        _validateOutflowIsAllowed();

        uint256 roundId = roundsCount;
        Round storage round = rounds[roundId];

        _validateRoundStatus(round, RoundStatus.Drawing);

        if (block.timestamp < round.drawnAt + 1 days) {
            revert DrawExpirationTimeNotReached();
        }

        round.status = RoundStatus.Cancelled;

        emit RoundStatusUpdated(roundId, RoundStatus.Cancelled);

        _startRound({_roundsCount: roundId});
    }
```

