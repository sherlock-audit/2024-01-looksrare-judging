Amusing Emerald Zebra

high

# Missing check for validity of timestamp

## Summary

The absence of a validity check for the message timestamp in the Yolo contract may allow the repeated use of an NFT floor price from the past, potentially compromising the games.

## Vulnerability Detail

In the Yolo contract, when receiving NFT floor price data from the Reservoir oracle, the contract verifies whether the signature's timestamp is stale. The current implementation checks if the `block.timestamp` exceeds the `floorPrice.timestamp` by a set `validity period`.

        function _verifyReservoirSignature(address collection, ReservoirOracleFloorPrice calldata floorPrice) private view {
            if (block.timestamp > floorPrice.timestamp + uint256(signatureValidityPeriod)) {
                revert SignatureExpired();
            }

https://github.com/sherlock-audit/2024-01-looksrare/blob/314fd5b753359e07778724342de8dda20713ec2e/contracts-yolo/contracts/YoloV2.sol#L1600-L1603

However, it lacks a crucial check to ensure that the `floorPrice.timestamp` is not in the future (if message.timestamp <= block.timestamp).

We can see the correct checking in the code provided by Reservoir:

        // Ensure the message timestamp is valid
        if (
            message.timestamp > block.timestamp ||
            message.timestamp + validFor < block.timestamp
        ) {
            return false;
        }

https://github.com/reservoirprotocol/oracle/blob/main/contracts/ReservoirOracle.sol#L49-L55

In known issue, the floor prices are expected to be trusted, but not the message.timestamp. The timestamp should be sufficiently verified.

## Impact

In case the Reservoir Oracle provides an error timestamp far in the future, the price value can be used over and over again with the wrong price until the future time becomes less than block.timestamp - validFor. Wrong price is a serious issue since it is used to price the NFT collections in the Yolo game.

## Code Snippet

https://github.com/sherlock-audit/2024-01-looksrare/blob/314fd5b753359e07778724342de8dda20713ec2e/contracts-yolo/contracts/YoloV2.sol#L1600-L1603

## Tool used

Manual Review

## Recommendation

Implement an additional check for the validity of `message.timestamp`:

```diff
-            if (block.timestamp > floorPrice.timestamp + uint256(signatureValidityPeriod)) {
+            if (block.timestamp > floorPrice.timestamp + uint256(signatureValidityPeriod)) || floorPrice.timestamp > block.timestamp {
                revert SignatureExpired();
            }
```