Howling Cloud Goldfish

medium

# Game is vulnerable to fee griefing attack

## Summary

There is a game-theoretic problem in Yolo v2 that allows an attacker to grief the protocol at a ratio close to 1:8(while I'm writing).

## Vulnerability Detail

I think there is a game theory issue in the protocol design: On each round that should be cancelled, an attacker can keep the game going by making two minimal deposit(using different addresses), causing the project to fall into a loss in the round.

Here is the proof of concept:

Firstly, let's consider costs other than gas costs.

On every VRF request, [Chainlink](https://docs.chain.link/vrf/v2/estimating-costs?network=ethereum-mainnet) would charge 0.68 $LINK:(not including callback gas)

![image](https://github.com/sherlock-audit/2024-01-looksrare-RealLTDingZhen/assets/156334774/ee7e9c64-cb41-4d4b-bf4a-57859d448349)

But, if the attacker chooses to pay fees in $LOOKS, the protocol would receive 2.5% of the total deposits for the round, which is `(0.01+0.01)*2.5% = 0.0005 $ETH`

So, on every round like this, protocol would lose `0.68 $LINK - 0.0005 $ETH = 8.422 $USD` (As I write this, the price of `$LINK` is 14 USD and the price of `$ETH` is 2196USD). Meanwhile the attacker loses `0.0005 $ETH = 1.098 $USD'.

If Looksrare expects fee income to be greater than VRF expenses, `LINK/WETH` must be a value above `0.0005/0.68 = 0.0007353` . As the chart below shows, since 2022, `LINK/WETH` is below the desired value until now.

![image](https://github.com/sherlock-audit/2024-01-looksrare-RealLTDingZhen/assets/156334774/a66ed3ac-8cd7-4f4d-8677-6281c1b6e526)

Whats more, since there are only two deposits in the round, protocol would also pay gas for `drawWinner` and `fulfillRandomWords`, which makes the loss bigger.

If we use statistics from [Yolo v1](https://looksrare.org/yolo/history), we can see that this attack can cause huge damage to the protocol, since a third of the rounds were cancelled!

## Impact

protocol is vulnerable to griefing attack, attacker can force fulfill a round that was supposed to be cancelled, and attacker losses are much smaller than protocol losses.

## Code Snippet

https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L401

## Tool used

Manual Review

## Recommendation

I think there should be a constraint on minimum deposit count to forbid such attack.