Rapid Ash Meerkat

medium

# Oracle Reliance

## Summary
The contract relies on external oracles for certain data, such as pricing information. If these oracles are manipulated or fail, the contract might use incorrect data, leading to flawed calculations or decisions.
## Vulnerability Detail
The contract interacts with price oracles (erc20Oracle, reservoirOracle). These oracles provide critical data used in the contract's calculations and logic. However, reliance on external data sources introduces risks, including the oracle being compromised, manipulated, or becoming unavailable.

## Impact
Incorrect data from an oracle can lead to incorrect payouts, token valuations, or other critical contract actions. This could result in financial loss for users and damage the contract's credibility.

## Code Snippet
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1162-L1165

## Tool used

Manual Review

## Recommendation

1. Use Decentralized Oracles: Prefer decentralized oracles with multiple independent sources to mitigate the risk of manipulation or failure.
2. Implement Oracle Failover Mechanisms: Have backup oracles or a mechanism to switch oracles if the primary oracle fails.
3. Validate Oracle Data: Implement checks to validate oracle data against sudden, unrealistic changes in values.
4. Monitor Oracle Health: Regularly monitor the health and integrity of the oracle data.