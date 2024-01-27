Jovial Obsidian Otter

high

# `updateValuePerEntry` doesn't check for already deposited funds in future rounds

## Summary
Function `depositETHIntoMultipleRounds` enables users to deposit in future rounds, while admin function that changes value of entry to a round does not check if some funds were deposited in future rounds, which may lead to unfairness in two ways:

1) A call to admin function `updateValuePerEntry` may be frontrunned to gain benefit in future rounds if `valuePerEntry` is changed to a value higher than the current one.

2) If in the call to `updateValuePerEntry` `valuePerEntry` is changed to a value lower than the current one and there are deposits to future rounds, game becomes unfair.

## Vulnerability Detail

As function `depositETHIntoMultipleRounds` allows users to deposit ETH in the current and future rounds, two scenarios are possible:

### Frontrun scenario
Admin calls `updateValuePerEntry` changing `valuePerEntry` to a higher value -> Malicious user frontruns the transaction, depositing ETH to future rounds using `depositETHIntoMultipleRounds`. As admin's call is frontrunned and `valuePerEntry` is not changed yet, malicious user's entries will be calculated at a lower `valuePerEntry` rate than future deposits made by other users. As `entriesCounts` value is calculated at the time of deposit, then the malicious user will gain unfair benefit.

For example, current `valuePerEntry` is 0.01 ETH, admin calls `updateValuePerEntry` setting `valuePerEntry` to 0.02 ETH. Malicious user frontruns the transaction and deposits 1 ETH to a future round. After that the admin's transaction is executed. Other user deposits 1 ETH to the same round, but their winning chances are twice lower than malicious user's. Hence, game becomes unfair.

###  `valuePerEntry` is changed to a lower value scenario
Some user deposits funds to several rounds using `depositETHIntoMultipleRounds`, then admin calls `updateValuePerEntry` and lowers `valuePerEntry` before all the rounds to which deposits were made are drawn. As the user has deposited funds at a higher `valuePerEntry` level, all players who deposit funds after the admin's transaction will get benefit compared to the user. Hence, game becomes unfair.

## Impact

In both scenarios, game may become unfair and some players will gain advantage by having higher winning chances compared to other players, even if they deposited the same amount of funds. This may lead to loss of funds.

## Code Snippet

```solidity
 function updateValuePerEntry(uint96 _valuePerEntry) external {
      _validateIsOwner();
      _updateValuePerEntry(_valuePerEntry);
  }
```

```solidity
  function _updateValuePerEntry(uint96 _valuePerEntry) private {
      if (_valuePerEntry == 0) {
          revert InvalidValue();
      }
      valuePerEntry = _valuePerEntry;
      emit ValuePerEntryUpdated(_valuePerEntry);
  }
```

function `updateValuePerEntry`:
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L795-L798

function `_updateValuePerEntry`:
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L871-L877

function `depositETHIntoMultipleRounds`:
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L312-L362

## Tool used

Manual Review

## Recommendation

I suggest to modify the `updateValuePerEntry` function so it checks whether deposits to future rounds exist and if they do, cancels those rounds.
