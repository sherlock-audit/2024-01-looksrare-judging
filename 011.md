Acrobatic Flint Lobster

medium

# Poor Trade Execution: Excess ERC-721 value not modulo `round.valuePerEntry` is lost when making a deposit.

## Summary

When depositing ERC-721s, the reservoir-defined floor price of these tokens is used to fairly determine the [`entriesCount`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1102) which can be allocated to the caller's YOLO position on an even playing around with native ether.

The floor price is non-deterministic, and is instead computed and cached dynamically for each round.

When the floor price of an asset is greater than [`valuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1102), excess value is not fairly compensated in competition entries, as these can only support integer denominations of the [`valuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1102). This has the implicit effect of rounding down token value when making a deposit, which in some unfortunate (though perfectly plausible) scenarios can incur a significant loss for the caller.

## Vulnerability Detail

When depositing native ether, [`YoloV2`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol) is meticulous in its assurance that callers cannot submit non-integer multiples of [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051) as their `msg.value` [under any circumstances](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1052):

```solidity
if (msg.value == 0) {
    if (deposits.length == 0) {
        revert ZeroDeposits();
    }
} else {
    uint256 roundValuePerEntry = round.valuePerEntry;
@>  if (msg.value % roundValuePerEntry != 0) {
        revert InvalidValue();
    }
    uint256 entriesCount = msg.value / roundValuePerEntry;
    totalEntriesCount += entriesCount;

    // ...
}
```

The intention here is that we do not wish for callers to inadvertently lose excess value on their native ether which cannot be equivalently redeemed in integer increments of share value.

However, there is no such assurance provided when transacting using ERC-721s, no doubt due to the complexities of and non-determinism of referencing the floor price:

```solidity
if (price == 0) {
    price = _getReservoirPrice(singleDeposit);
    prices[tokenAddress][roundId] = price;
}

uint256 entriesCount = price / round.valuePerEntry; /// @audit round_down
if (entriesCount == 0) {
    revert InvalidValue();
}
```

This leads to the scenario where the floor price of a supported token is _greater_ than [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051) but _less than_ the price of `2 * roundValuePerEntry`, where the difference between these two values is simply abandoned with no tangible increase to win probability.

It is a given that users indeed agree to yield to the floor price of a collection when YOLOing their non-fungible assets, however it is perfectly reasonable to assume a user would do this on the condition their token value would be reflected fairly and compensated in an equivalent number of YOLO entries, which in such a scenario, they are not. Instead, they both incur a loss and join the contest at a relative disadvantage when compared to liquid users who were able to deposit using native ether.

## Impact

I am inclined to assess this as a medium-severity issue due to both the potential for loss of funds and pronounced likelihood of occurrence, since the floor price of a collection is newly-determined for each round, and will _never_ be an integer multiple of [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051).

This makes it very likely to [latch](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1097C21-L1100C22) at an economically-inefficient point in time.

Further, if LooksRare anticipate to launch a high-stakes form of YOLO (consisting of high values of [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051) and allowlisted blue-chip NFTs), it could be argued that this is a high severity issue.

## Code Snippet

```solidity
if (singleDeposit.tokenType == YoloV2__TokenType.ERC721) {
    if (price == 0) {
        price = _getReservoirPrice(singleDeposit);
        prices[tokenAddress][roundId] = price;
    }

    uint256 entriesCount = price / round.valuePerEntry;
    if (entriesCount == 0) {
        revert InvalidValue();
    }

    uint256[] memory amounts = new uint256[](singleDeposit.tokenIdsOrAmounts.length);
    for (uint256 j; j < singleDeposit.tokenIdsOrAmounts.length; ++j) {
        totalEntriesCount += entriesCount;

        if (currentEntryIndex != 0) {
            currentEntryIndex += uint40(entriesCount);
        } else {
            currentEntryIndex = _getCurrentEntryIndexWithoutAccrual(
                round,
                roundDepositCount,
                entriesCount
            );
        }

        uint256 tokenId = singleDeposit.tokenIdsOrAmounts[j];

        // tokenAmount is in reality 1, but we never use it and it is cheaper to set it as 0.
        // This is equivalent to
        // round.deposits.push(
        //     Deposit({
        //         tokenType: YoloV2__TokenType.ERC721,
        //         tokenAddress: tokenAddress,
        //         tokenId: tokenId,
        //         tokenAmount: 0,
        //         depositor: msg.sender,
        //         withdrawn: false,
        //         currentEntryIndex: currentEntryIndex
        //     })
        // );
        // unchecked {
        //     roundDepositCount += 1;
        // }
        uint256 depositDataSlotWithCountOffset = _getDepositDataSlotWithCountOffset(
            roundDepositsLengthSlot,
            roundDepositCount
        );
        _writeDepositorAndCurrentEntryIndexToDeposit(depositDataSlotWithCountOffset, currentEntryIndex);
        _writeTokenAddressToDeposit(
            depositDataSlotWithCountOffset,
            YoloV2__TokenType.ERC721,
            tokenAddress
        );
        assembly {
            sstore(add(depositDataSlotWithCountOffset, DEPOSIT__TOKEN_ID_SLOT_OFFSET), tokenId)
            roundDepositCount := add(roundDepositCount, 1)
        }

        amounts[j] = 1;
    }

    batchTransferItems[i].tokenAddress = tokenAddress;
    batchTransferItems[i].tokenType = TransferManager__TokenType.ERC721;
    batchTransferItems[i].itemIds = singleDeposit.tokenIdsOrAmounts;
    batchTransferItems[i].amounts = amounts;
}
```

## Tool used

Vim, Foundry

## Recommendation

There are a number of possible recommendations which could be considered to minimize the impact of floor values which are significantly in excess of [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051):

1. Consider introducing a form of slippage protection. If excess evaluated floor price of a collection is too great a ratio of [`roundValuePerEntry`](https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1051), an attempt to deposit should `revert`.
2. Use decimal precision for shares to more accurately represent share ownership.