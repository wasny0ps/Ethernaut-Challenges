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
