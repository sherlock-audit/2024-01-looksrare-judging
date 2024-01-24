Elegant Raisin Woodpecker

high

# VRF block confirmation time of 3 is not sufficient on certain chains and reorgs can change the winner of a round

## Summary
VRF block confirmation time of 3 is not sufficient on certain chains that often have reorg with a depth > 3.
Reorgs can change the winner of a round.

## Vulnerability Detail
This contract could potentially be deployed to any EVM compatible L2. In certain chains, a block confirmation time of 3 will not be sufficient. 

For example, on Polygon, reorg with a depth of more than 3 happen very often, previously there was a reorg with a depth of 157.
https://polygonscan.com/blocks_forked

The block confirmation time is how many blocks VRF waits before fulfilling the randomness request.

If there is a reorg with a depth more than the block confirmation time, the function call to `drawWinner()` which requests randomness can be included in a different block. This would result in a different VRF output and potentially a different winner.

## Impact
If a reorg occurs, the function call to `drawWinner()` which requests randomness can be included in a different block. 
This would result in a different VRF output and potentially a different winner. 
Consequently, the original winner would lose their prize.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1004

## Tool used
Manual Review

## Recommendation
Instead of hard-coding the block confirmation time to 3, it should be set differently depending on the blockchain that it is being deployed to

https://docs.chain.link/vrf/v2/security#choose-a-safe-block-confirmation-time-which-will-vary-between-blockchains
