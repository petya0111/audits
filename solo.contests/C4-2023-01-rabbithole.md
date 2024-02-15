[RabbitHole contest](https://code4rena.com/reports/2023-01-rabbithole)
# [H-01] Unprotected methods due wrong expression in onlyMinter modifiers leads to access control vulnerability 
https://github.com/code-423n4/2023-01-rabbithole-findings/issues/227 
## Impact 
The modifier `onlyMinter` that is declared in `RabbitHoleReceipt.sol` and `RabbitHoleTickets.sol` files has no impact and has no protection for methods that are used.

The modifier `onlyMinter` is used in 3 functions. Having no right check would mean that this modifier will always be bypassed and would result in everyone having the ability to call the mint and mintBatch functions. I presume that the used in two files is a mistyped error and also copy-pasted in the other file. This can lead to more problems if more modifiers are developing like that.

## Proof of Concept
File: contracts/RabbitHoleReceipt.sol

```solidity
58:     modifier onlyMinter() {
59:        msg.sender == minterAddress;
60:        _;
61      }
```
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleReceipt.sol#L58-L61

File: contracts/RabbitHoleTickets.sol

```solidity
47:     modifier onlyMinter() {
48:        msg.sender == minterAddress;
49:        _;
50:     }
```
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/RabbitHoleTickets.sol#L47-L50


## Tools Used

No tools were used only I spotted it like the wrong expression.

## Recommended Mitigation Steps

It needs to be written for example like`require` a rule 
`require(msg.sender == minterAddress, “The sender must match the minter address”)`

Or be a custom error with revert 
`if(msg.sender != minterAddress) revert DifferentMinterAddress()`