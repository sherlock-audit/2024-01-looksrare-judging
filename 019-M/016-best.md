Mythical Satin Tuna

medium

# Protocol fees can be more than potential gains

## Summary
Winner claims all the deposits that's been deposited in a round. Some percentage of the total deposits are taken as protocol fee and has to be paid to the protocol upon claiming. However, in some cases, the protocol fee to be paid can be higher than the winners gains. 
## Vulnerability Detail
Assuming the "roundValuePerEntry" for a round is 1 ether, means that participants need to pay 1 ether each to join (assuming no one uses NFTs or ERC20 tokens), let's consider a specific scenario for a given round:

Alice deposits 1 ether, resulting in her entry index being 1.
Bob deposits 1 ether, making his entry index 1 + 1 = 2.
At this point, there are a total of 2 ethers in the pool. Subsequently, Carol, observing the growing prize pool, decides to go all-in.

Carol deposits 100 ethers, causing his entry index to become 2 + 100 = 102.
Now, with a total of 102 entries and 102 ethers, the round concludes, and Carol emerges as the winner, given his substantial deposit compared to Alice and Bob.

Upon invoking the drawWinner function, the "protocolFee" for the round is calculated as follows:
```solidity
round.protocolFeeOwed = (round.valuePerEntry * currentEntryIndex * round.protocolFeeBp) / 10_000;
```

In the code snippet above, considering that the protocolFeeBp is set to "300" in the test file, and the currentEntryIndex is 102, the protocolFeeOwed is computed as:
1  * 102 * 300 / 10_000 = 3.06 ether

That means Carol deposited 100 ether, won 2 more ether (Alice and Bob's) and has to pay 3.06 ether as protocol fee which in theory that means Carol lost 3.06 - 2 = 1.06 ether. 

Although Carol won, Carol actually lost in terms of ether.

We can say that Carol needed to check his potential gains before depositing to escalate this issue. However, Carol could also have deposited via the `depositETHIntoMultipleRounds` function, where it auto-fills the ETH for future rounds. Since Carol has no idea what the future rounds' total deposits will be, he will always have the chance to lose money because of the protocol fees.

Assume Carol deposits 1 ether which is relatively very less amount and he deposits it for the upcoming 10 rounds (total 10 ether). Then Alice and Bob deposits 0.01 ether hence, the total pod will be 1.02 ether for the current round. The protocol fee to be charged will be 1.02 * 300 / 10_000 = 0.0306 ether which is bigger than Alice and Bobs deposits! 
## Impact
This could make the life of big depositors very hard since they have to calculate the protocol fee before they deposit. Also since this will disincentivize big depositors the smaller depositors earning will be capped. It might give a fair advantage for smaller depositors tho, so they make sure someone is not going to deposit a very large amount and win the game 99%. However, this is a game of luck and this behaviour should be possible and should still be fine imo. Also, since anyone can deposit via depositETHIntoMultipleRounds to future rounds where there is no info who deposits how much players will hesitate depositing before hand. After these considerations, I am not sure whether to submit this as a medium or not but imo it can be a medium so I'll create the issue.
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1270-L1296
## Tool used

Manual Review

## Recommendation