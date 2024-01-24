Sharp Indigo Ostrich

medium

# Users can bypass outflow gate via calling rolloverEth()

## Summary

The outflowAllowed flags allows the owner of the contract to restrict whether or not tokens are transferred from the contract. This flag when set to false will prevent the following contracts from being called:

- cancel()
- cancelAfterRandomnessRequest()
- claimPrizes() 
- withdrawDeposits()

However, this flag is not checked when calling rolloverEth(), which in an edge case will send tokens to the depositor who is rolling over ETH into a new round, even if outflow is disabled. 

## Vulnerability Detail

Yolo allows the owner to toggle the outflowAllowed flag, which dictates whether depositors can withdraw tokens from the contract. 

When calling rolloverEth(), if a depositor is rolling over their deposit into the current round, then the following code will be called inside rolloverEth():

```solidity
uint256 roundValuePerEntry = round.valuePerEntry;
uint256 dust = rolloverAmount % roundValuePerEntry;
if (dust != 0) {
    unchecked {
        rolloverAmount -= dust;
    }
    // AUDIT: no check here that outflow is disabled
    _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, dust, gasleft());
}
```

This code checks whether or not the rollover amount is above the current round's entry value. If the rollover amount is above the entry amount, then the leftover ETH will be transferred to the depositor. There are no validations here checking if outflowAllowed is disabled. 

Below is a forge test showing a simple example where rollbackEth() is called and a user receives a direct transfer refund from the protocol.

```solidity
function test_rolloverETH_OutflowAllowedBypass() public {
    vm.deal(user1, 0.1 ether);
    vm.deal(user2, 1 ether);

    vm.prank(owner);
    // AUDIT: toggle outflow is now disabled
    yolo.toggleOutflowAllowed();

    assertEq(yolo.outflowAllowed(), false);

    vm.prank(user1);
    yolo.deposit{value: 0.1 ether}(1, _emptyDepositsCalldata());

    vm.prank(user2);
    yolo.deposit{value: 1 ether}(1, _emptyDepositsCalldata());

    _drawRound();

    uint256[] memory randomWords = new uint256[](1);
    randomWords[0] = 110;

    vm.prank(VRF_COORDINATOR);
    VRFConsumerBaseV2(yolo).rawFulfillRandomWords(FULFILL_RANDOM_WORDS_REQUEST_ID, randomWords);

    IYoloV2.WithdrawalCalldata[] memory withdrawalCalldata = _singleRoundRolloverWithdrawalData();

    vm.prank(user1);
    yolo.rolloverETH(withdrawalCalldata, false);

    // AUDIT: user received ETH when they shouldn't have
    assertEq(user1.balance, 0.007 ether);
}
```

## Impact

Depositors will still be able to retrieve some tokens from the protocol bypassing the outflowAllowed flag.

The chances of this bug occurring are moderately low because this bug requires:

- the currentRound have a smaller value per entry than previous entries
- only the delta between the current round entry value and the rollover amount will be sent to the depositor
- only occurs when outflowAllowed is disabled

However, because this breaks a critical feature in the protocol designed to stop any funds from being withdrawn from the contract, I believe this is a MED.

## Code Snippet

https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol?plain=1#L724-L730

## Tool used

Manual Review

## Recommendation

Protocol should revert if rolloverEth() is going to distribute leftover tokens to the depositor.

```solidity
// Inside rolloverEth()

uint256 roundValuePerEntry = round.valuePerEntry;
uint256 dust = rolloverAmount % roundValuePerEntry;
// AUDIT: require check added below to ensure dust can't be transferred out of contract.
require(dust != 0 && _validateOutflowIsAllowed(), "Can't rollover ETH while outflow is disabled");
if (dust != 0) {
    unchecked {
        rolloverAmount -= dust;
    }
    _transferETHAndWrapIfFailWithGasLimit(WETH, msg.sender, dust, gasleft());
}
```
