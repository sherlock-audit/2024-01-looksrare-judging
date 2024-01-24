Jumpy Burlap Mockingbird

medium

# YoloV2 is not Gelato compatible and lack RNGLib Integration as recommended

## Summary
The current implementation of the smart contract's randomness generation for winner selection is incompatible with Gelato VRF, as it does not adhere to Gelato's VRF integration guidelines and lacks the use of RNGLib. 

## Vulnerability Detail
Gelato functions are different from those in chainlink VRF : 
`requestRandomWords` <=> `_requestRandomness`
`fulfillRandomWords` <=> `fulfillRandomness`

According to the Gelato VRF documentation, the recommended approach for randomness is to inherit from `GelatoVRFConsumerBase.sol`, which utilizes `RNGLib`. This library allows for dynamic fetching of random values and provides additional security, especially crucial when multiple applications operate simultaneously.

The current contract does not inherit from `GelatoVRFConsumerBase.sol`, nor does it use `RNGLib`.

## Impact
There is an easy to see but clear non-operability with non chainlink VRF L2s stated to be compatible with in the ReadMe
> Mainnet Arbitrum Base Potentially any EVM compatible L2 (Including ZK L2s)
> Gelato VRF is used to fulfill randomness on Base

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1270

## Tool used

Manual Review

## Recommendation

Adapt a second contract to Gelato VRF