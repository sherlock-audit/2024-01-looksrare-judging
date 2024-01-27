Recumbent Hemp Chameleon

high

# YoloV2::withdrawDeposits() will send extra ERC20 tokens to the withdrawing user or revert

## Summary
Withdraw Deposits() has a buggy logic that will result in sending of extra tokens to the caller. Incase the YoloV2 contract does not have sufficient tokens in the balance, the transaction will revert.

## Vulnerability Detail
withdrawDeposits() function takes an array of tokens that can be withdrawn from a round that was cancelled.
During deposit() call, user can deposit ERC20 or ERC721. So, the singledesposit in the round can contain both ERC20 and ERC721.

The withdraw Logic for ERC20 uses a memory structure call "TransferAccumulator" which can track amount for a single ERC20 token. While looping through the deposits, in order to transfer the tokens, the logic uses _transferTokenOut() function to transfer the tokens to the msg.sender from the contract's balance.

For ERC721, it is clean transfer.

For ERC20, in the _transferTokenOut(), if the memory variable as the same token address as the current singledeposit, it will accumulate, else it will perform the transfer using the _executeERC20DirectTransfer() function. 

```solidity
  _executeERC20DirectTransfer(
                        transferAccumulator.tokenAddress,
                        msg.sender,
                        transferAccumulator.amount
                    );
```

Post transfer, it updates the memory variable again with token address  and token amount.

```solidity
     transferAccumulator.tokenAddress = tokenAddress;
     transferAccumulator.amount = singleDeposit.tokenAmount;
```
The problem is, once the call exists the single Deposit loop, if the transferAccumulator.amount is greater than 0, it again transfer the token balance the second time, sending out duplicate tokens from the YoloV2 contract.

Incase, there is no balance for such tokens in YoloV2 contract, then the transaction will revert.

lets say, in a round, these are the three tokens.
    
Deposits = [{token:tokenA, amont:100},{token:tokenB, amont:200},{token:tokenB, amont:150}];
   
transferAccumulator is memory variable that is declared and not initialised.

Now, in the loop on deposits,
**[loop:0 == tokenA]** enter _transferTokenOut, at this point, **transferAccumulator.tokenAddress is empty** and hence
                                    it will enter the else condition and transfer using _executeERC20DirectTransfer().
                                    It will also update the **transferAccumulator::tokenAddress** **to tokenA** and amount to 100.

**[loop:1 == tokenB]** enter _transferTokenOut, at this point, transferAccumulator.tokenAddress is tokenA and hence
                                    it will enter the else condition and transfer using _executeERC20DirectTransfer().
                                    It will also update the **transferAccumulator::tokenAddress** **to tokenB** and amount to 200.

**[loop:2 == tokenC]** enter _transferTokenOut, at this point, transferAccumulator.tokenAddress is tokenB and hence
                                    it will enter the else condition and transfer using _executeERC20DirectTransfer().
                                    It will also update the **transferAccumulator::tokenAddress** **to tokenC** and amount to 150.


Now, once it returns back to withdrawDeposits(), the transferAccumulator will have the below info

  transferAccumulator.tokenAddress = tokenC;
  transferAccumulator.amount = 150;

since amount is not 0, it will try to send 150 tokens of tokenC to the caller function again.


## Impact
Broken token tracking system as tokens of one user could be transferred to another user while some users cannot claim the tokens back at all.

## Code Snippet
_transferTokenOut function where post transfer, it records the tokenAddress and amount.
https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L1463-L1496

Below is the logic where post transfer, the token address and amount are updated in transferAccumulator object.

```solidity
} else if (tokenType == YoloV2__TokenType.ERC20) {
            address tokenAddress = singleDeposit.tokenAddress;
            if (tokenAddress == transferAccumulator.tokenAddress) {
                transferAccumulator.amount += singleDeposit.tokenAmount;
            } else {
                if (transferAccumulator.amount != 0) {
                    _executeERC20DirectTransfer(
                        transferAccumulator.tokenAddress,
                        msg.sender,
                        transferAccumulator.amount
                    );
                }

                transferAccumulator.tokenAddress = tokenAddress;
                transferAccumulator.amount = singleDeposit.tokenAmount;
            }
        }
```

Also, refer to the  withdrawDeposits() function where on returning, it is sending the tokens again.

https://github.com/sherlock-audit/2024-01-looksrare/blob/main/contracts-yolo/contracts/YoloV2.sol#L625-L631

and here, when transferAccumulator.amount !=0, it attempts to send tokens again.
```solidity
if (transferAccumulator.amount != 0) {
            _executeERC20DirectTransfer(transferAccumulator.tokenAddress, msg.sender, transferAccumulator.amount);
        }
```

## Tool used
Manual Review

## Recommendation
The problem is in how the logic is implemented. It is trying to transfer in two places, once in the main function and again in the _transferTokenOut function.

Segregating the identification logic to account the tokens to be transfer in one place and then separating the actual transfer in a clean function will address this issue.