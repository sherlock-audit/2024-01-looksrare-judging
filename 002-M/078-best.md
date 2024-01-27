Mysterious Seafoam Huskie

medium

# The number of deposits in a round can be larger than MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND

## Summary
The number of deposits in a round can be larger than [MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L81), because there is no such check in [depositETHIntoMultipleRounds()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L312) function or [rolloverETH()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L643-L646) function.

## Vulnerability Detail
**depositETHIntoMultipleRounds()** function is called to deposit ETH into multiple rounds, so it's possible that the number of deposits in both current round and next round is **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**.

When current round's number of deposits reaches **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**, the round is drawn:
```solidity
        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
```
[_drawWinner()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L997) function calls VRF provider to get a random number, when the random number is returned by VRF provider, [fulfillRandomWords()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1270) function is called to chose the winner and the next round will be started:
```solidity
                _startRound({_roundsCount: roundId});
```
If the next round's deposit number is also **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**, [_startRound()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L949) function may also draw the next round as well, so it seems that there is no chance the the number of deposits in a round can become larger than **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**:
```solidity
            if (
                !paused() &&
                _shouldDrawWinner(numberOfParticipants, round.maximumNumberOfParticipants, round.deposits.length)
            ) {
                _drawWinner(round, roundId);
            }
```
However, **_startRound()** function will draw the round **only if the protocol is not paused**. Imagine the following scenario:
1. The deposit number in `round 1` and `round 2` is **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**;
2. `round 1` is drawn, before random number is sent back by VRF provider, the protocol is paused by the admin for some reason;
3. Random number is returned and **fulfillRandomWords()** function is called to start `round 2`;
4. Because protocol is paused, `round 2` is set to **OPEN** but not drawn;
5. Later admin unpauses the protocol, before [drawWinner()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L401) function can be called, some users may deposit more funds into `round 2` by calling **depositETHIntoMultipleRounds()** function or **rolloverETH()** function, this will make the deposit number of `round 2` larger than **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**.

Please run the test code to verify:
```solidity
    function test_audit_deposit_more_than_max() public {
        address alice = makeAddr("Alice");
        address bob = makeAddr("Bob");

        vm.deal(alice, 2 ether);
        vm.deal(bob, 2 ether);

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 0.01 ether;
        amounts[1] = 0.01 ether;

        // Users deposit to make the deposit number equals to MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND in both rounds
        uint256 MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND = 100;
        for (uint i; i < MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND / 2; ++i) {
            vm.prank(alice);
            yolo.depositETHIntoMultipleRounds{value: 0.02 ether}(amounts);

            vm.prank(bob);
            yolo.depositETHIntoMultipleRounds{value: 0.02 ether}(amounts);
        }

        // owner pause the protocol before random word returned
        vm.prank(owner);
        yolo.togglePaused();

        // random word returned and round 2 is started but not drawn
        vm.prank(VRF_COORDINATOR);
        uint256[] memory randomWords = new uint256[](1);
        uint256 randomWord = 123;
        randomWords[0] = randomWord;
        yolo.rawFulfillRandomWords(FULFILL_RANDOM_WORDS_REQUEST_ID, randomWords);

        // owner unpause the protocol
        vm.prank(owner);
        yolo.togglePaused();

        // User deposits into round 2
        amounts = new uint256[](1);
        amounts[0] = 0.01 ether;
        vm.prank(bob);
        yolo.depositETHIntoMultipleRounds{value: 0.01 ether}(amounts);

        (
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            YoloV2.Deposit[] memory round2Deposits
        ) = yolo.getRound(2);

        // the number of deposits in round 2 is larger than MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND
        assertEq(round2Deposits.length, MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND + 1);
    }
```

## Impact
This issue break the invariant that the number of deposits in a round can be larger than **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L312
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L643-L646

## Tool used
Manual Review

## Recommendation
Add check in [_depositETH()](https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1417-L1422) function which is called by both **depositETHIntoMultipleRounds()** function and **rolloverETH()** function to ensure the deposit number cannot be larger than **MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND**:
```diff
        uint256 roundDepositCount = round.deposits.length;

+       if (roundDepositCount >= MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND) {
+           revert MaximumNumberOfDepositsReached();
+       }

        _validateOnePlayerCannotFillUpTheWholeRound(_unsafeAdd(roundDepositCount, 1), round.numberOfParticipants);
```