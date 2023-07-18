<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel14.svg">


# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

Challenge's message:

> This gatekeeper introduces a few new challenges. Register as an entrant to pass this level. - Remember what you've learned from getting past the first gatekeeper the first gate is the same. - The assembly keyword in the second gate allows a contract to access functionality that is not native to vanilla Solidity. See [here](https://docs.soliditylang.org/en/v0.4.23/assembly.html) for more information. The extcodesize call in this gate will get the size of a contract's code at a given address - you can learn more about how and when this is set in section 7 of the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf). - The ^ character in the third gate is a bitwise operation (XOR), and is used here to apply another common bitwise operation (see [here](https://docs.soliditylang.org/en/v0.4.23/miscellaneous.html#cheatsheet)). The Coin Flip level is also a good place to start when approaching this challenge.


The *GatekeeperTwo* contract implements a gatekeeping mechanism that restricts entry to a certain function based on three specific conditions. There is a public address entrant variable that stores the address of the entrant who successfully passes the gatekeeping conditions.

### Modifiers

#### gateOne()

This modifier enforces a condition that the *msg.sender* (the caller of the function) is not the same as the *tx.origin* (the origin of the transaction). In other words, **it ensures that the function is not called directly but is being called through another contract or function**.

#### gateTwo()

This modifier checks if the caller of the function has an external code size of zero. It does this by using **inline assembly** to **retrieve the external code size of the caller's address**. If the code size is zero, the function execution proceeds, and if not, it throws an error. **This condition prevents the function from being called by another contract**.

#### gateThree()

This modifier takes a parameter *_gateKey* of type *bytes8* and enforces one condition:

The condition checks if the XOR operation between the first 8 bytes of the keccak256 hash of the *msg.sender* address and the *_gateKey* is equal to the maximum value of type *uint64*. If this condition is not met, the function execution throws an error. **This condition introduces a cryptographic check on the *_gateKey* to verify the authenticity of the entrant**.

In the enter() function is the entry point of the contract and is used to gain access based on the gatekeeping conditions. It takes a parameter _gateKey of type bytes8 which is used in the gateThree modifier. The function first applies the *gateOne*, *gateTwo*, and *gateThree* modifiers to enforce the respective conditions. If all the conditions are met, the *entrant* variable is updated with the tx.origin (the origin of the transaction) and the function returns *true*.

It's important to note that the code uses low-level assembly and cryptographic operations to perform the gatekeeping checks, which may have been implemented for specific security requirements or anti-cheat mechanisms. The exact rationale behind the conditions is not provided in the code, and further context would be necessary to understand the full purpose and security implications of the gatekeeping mechanism.

## Inner Workings Of Contract Creation

The inner workings of contract creation on Ethereum involve several key components and steps. Here's a more detailed explanation of the process:

1. **Compilation:** The contract code, written in a high-level language like Solidity, is compiled into bytecode. The bytecode represents the contract's instructions in a low-level format that can be executed on the Ethereum Virtual Machine (EVM).
2. **Transaction Initialization:** To create a contract, a special transaction called a "contract creation transaction" is initialized. This transaction has no recipient and contains the bytecode of the contract in its data field. It also includes the necessary transaction details like the sender's address, gas limit, and gas price.
3. **Contract Address Determination:** When the transaction is signed and broadcasted to the network, Ethereum nodes receive and process it. The transaction is then included in a block during the mining process. The address of the contract is determined based on the sender's address and the nonce of the sender's account. This ensures the uniqueness of the contract address.
4. **EVM Execution:** When a miner includes the contract creation transaction in a block, the EVM executes the contract creation process. It involves several steps:

   - **Gas and Fee Calculation:** The gas limit and gas price specified in the transaction determine the maximum amount of computational resources (gas) that can be used for contract creation. The sender must pay fees proportional to the gas used.
   - ** Account Creation:** A new account is created on the Ethereum network to represent the contract. The account is associated with the contract address determined earlier.
   - ** Contract Code Deployment:** The bytecode of the contract is stored in the contract account's code storage. This code represents the logic and functionality of the contract.
   - **Contract Initialization:** If the contract has a constructor function, it is executed after the code deployment. The constructor function performs any necessary initialization tasks, such as setting initial values or allocating resources.
   - **State and Storage Creation:** The contract account is assigned an initial state and storage, which includes variables and storage slots used by the contract during its execution.

5. **Gas Consumption and Fee Deduction:** During the contract creation process, each computational step consumes gas. The gas consumed is subtracted from the gas limit specified in the transaction, and the corresponding fee is deducted from the sender's account balance.
6. **Transaction Finalization:** Once the contract creation process is successfully completed, the transaction is considered finalized, and the contract is live on the Ethereum network. The contract can now receive transactions and execute its functions.

These steps outline the internal mechanics of contract creation on Ethereum. They involve compiling the code, initializing a contract creation transaction, determining the contract address, executing the EVM process, and finalizing the transaction. Understanding these inner workings is crucial for developers and users interacting with smart contracts on the Ethereum network.


If you want to more about this topic, you might save [ethereum yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf). Also, I should strongly recommend read [this article](https://medium.com/coinmonks/ethernaut-lvl-14-gatekeeper-2-walkthrough-how-contracts-initialize-and-how-to-do-bitwise-ddac8ad4f0fd).

## Bitwise Operations

Solidity supports the following logic gate operations:

- `&`: and(x, y) bitwise and of x and y; where 1010 & 1111 == 1010
- `|`: or(x, y) bitwise or of x and y; where 1010 | 1111 == 1111
- `^`: xor(x, y) bitwise xor of x and y; where 1010 ^ 1111 == 0101
- `~`: not(x) bitwise not of x; where ~1010 == 0101

In the XOR process, if `A xor B = C`, then `A xor C = B`.

# Subverting

In this section, we will bypass the modifiers one by one. For this reason, I split up this cases into to sub texts.

### gateOne()

This modifier requires the msg.sender is not the same as the tx.origin. That means, we will **always pass this condition because of our attack contract calls the enter() function externally**.


### gateTwo()

In this gate, we can see that how Solidity allow you to write code in a lower-level language called **Yul**. This is not the place to discuss this topic, but if you want to learn more, there are tons of content about Yul [from here.](https://docs.soliditylang.org/en/latest/yul.html)

- The ***CALLER*** opcode **returns the 20-byte address of the caller account**. This is the account that did the last call (except delegate call).
- The ***EXCODESIZE*** opcode do when executed **returns the code size in bytes of the address that is passed as parameter**.

This gate required that the code size of the caller must be 0. If the caller was an EOA (**Externally Owned Account**) that would always **return zero**, but this cannot be the case because as we said the caller (msg.sender) must be a smart contract because of the first gate requirement. 

A smart contract has two different byte codes when compiled:

- The **creation bytecode** is the bytecode needed by Ethereum to create the contract and execute the constructor only **once**.
- The **runtime bytecode** is the real code of the contract, the one stored in the blockchain and that will be used to execute your smart contract functions.

**When the constructor is executed initializing the contract storage, it returns the runtime bytecode**. Until the very end of the constructor the contract itself does not have any runtime bytecode, this mean that if you call `address(contract).code.length` it would return `0`!

If you want to read more about this at EVM level, you can have a deep dive into the OpenZeppelin blog post [Deconstructing a Solidity Contract — Part II: Creation vs. Runtime](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-ii-creation-vs-runtime-6b9d60ecb44c).

For this reason, to pass the second gate, we just need to call enter from the exploiter smart contract `constructor`!

### gateThree()

In the XORs there is a certain rule. If `A xor B = C`, then `A xor C = B`. So, when we look at the requirement in the gateThree modifier, we can find the *_gateKey* with changing places beetwen `type(uint64).max` and `uint64(_gateKey)` expressions.

```solidity
bytes8 gateKey = bytes8(keccak256(abi.encodePacked(address(this)))) ^ bytes8(type(uint64).max);
```

This command, calculate the gateKey with XOR beetwen `bytes8(keccak256(abi.encodePacked(address(this))))` and `bytes8(type(uint64).max`. Thus, we will find the gateKey correctly.

End of some bypassing staff, here is our attack contract.

```solidity
pragma solidity ^0.8.17;

import './GatekeeperTwo.sol';

contract Attack{

    GatekeeperTwo target;

    constructor(address _target){
        target = GatekeeperTwo(_target);
        bytes8 gateKey = bytes8(keccak256(abi.encodePacked(address(this)))) ^ bytes8(type(uint64).max);
        target.enter(gateKey);  
    }
}
```

Shortly, our attack contract gets an target contract's instance and call **enter()** function with the calculated gateKey in the constructor. Let's do it.

First of all, check the entrant variable's value from the target contract.

```shell
await contract.entrant()
'0x0000000000000000000000000000000000000000'
```

After attack contract deployment's process, we will register as an entrant to the level's contract. Check the entrant's value again. Here we go!

```shell
await contract.entrant()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

```shell
player
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

0age's,who is owner of the challenge, message:

> Way to go! Now that you can get past the gatekeeper, you have what it takes to join theCyber, a decentralized club on the Ethereum mainnet. Get a passphrase by contacting the creator on reddit or via email and use it to register with the contract at gatekeepertwo.thecyber.eth (be aware that only the first 128 entrants will be accepted by the contract).


**_by wasny0ps_**

