Mythical Satin Tuna

medium

# Game might not work in L2's

## Summary
In the Sherlock doc it states that the contracts will be deployed to several L2's including ARB. However, there are few things that will prevent contract to be used in ARB. 
## Vulnerability Detail
1- It can't be deployed because of the usage of Solidity 0.8.20+. Arbitrum does not support the PUSH0 opcode yet, so versions 0.8.20 and above won't be usable.

2- Even if the Solidity version is decremented to 0.8.19, the contract will be deployed normally, but the VFR will most likely not function due to the 500,000 gas limit being too low for Arbitrum. In Arbitrum, since the gas price is very low and the gas amount is high in their configuration, 500,000 will be insufficient for the randomness request call.

Let's elaborate a bit more on point 2 and illustrate a scenario where funds can be lost. The contract is deployed to ARB with a 500,000 gas limit supporting randomness requests. The first-ever round starts, and users deposit into the first round. When the round is completed, the winner is drawn, and Chainlink randomness requests are made. However, the Chainlink call to fulfillRandomWords might fail because 500,000 is too little. In such a scenario, the cancelAfterRandomnessRequest has to be called so that depositors can claim back their funds. Overall, the contract has to be redeployed with a gas limit higher than 500,000.
## Impact
As stated in the details section, the round has to be cancelled if such scenario happens and the funds will not be lost for the users. Only funds that will be lost is the owners re-deployment cost. However, since in the Sherlock doc it's clearly stated that this contract will be deployed in Arb mainnet I believe the current code should indeed needed to work in Arb mainnet without any changes. Hence, I'll label this as medium.

I wasn't able to run the tests in the arbitrum mainnet but I am very sure the gas required can be easily higher than 500_000. Here even a simple ERC20 approve tx cost is 300K+
https://arbiscan.io/tx/0xa4916ff6d697256f2e07e3a12007c9be13cfe541154957230b92b838f0363f29
## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/7d76b96a58a6aee38f23bb38b8a5daa3bdc03f7c/contracts-yolo/contracts/YoloV2.sol#L1001-L1007
## Tool used

Manual Review

## Recommendation
For every deployment set the callback gas limit in the constructor so that the every chain can have a proper gas limit respect to their configs. 