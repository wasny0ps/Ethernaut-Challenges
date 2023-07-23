# Target Contract Review

Given contact.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

Challenge's message: 

> To solve this level, you only need to provide the Ethernaut with a Solver, a contract that responds to whatIsTheMeaningOfLife() with the right number.

In this challenge, we are facing a basic smart contract which saves the solver's address with the **setSolver()** function. 

To pass the challenge, we must have some assembly programming to deploy the solver contract. And there are other requirements we will do it.


# Contract Creation

Contract creation in the Ethereum Virtual Machine (EVM) is the process of deploying a new smart contract onto the Ethereum blockchain. When a contract is created, a new address is generated for that contract, and its bytecode is stored on the blockchain. Here's how the contract creation process works in the EVM:

1. **Contract Compilation**
  - First, the developer writes the smart contract code in a high-level language like Solidity. The code is then compiled using a Solidity compiler, which generates the EVM bytecode and ABI. 
2. **Transaction Creation**
  - To deploy the contract, the developer creates a transaction, specifically a "contract creation transaction." This transaction is a special type of transaction that contains all the necessary information to create a new contract. The transaction includes:
    + An empty recipient address.
    + The **EVM bytecode** of the contract.
    + Any initial data or parameters required by the contract constructor.
    + Gas price and gas limit for the deployment.
3. **Contract Deployment**
  - When the transaction is processed, the EVM creates a new contract at a unique address. This address is derived from the sender's address and the sender's nonce.
4. **Contract Initialization**
  - If the contract has a constructor function, it is executed during the contract creation process. The constructor sets up initial values and state variables for the contract from the **contract creation bytecode**. Contract creation bytecode involves both **initialization code** and the **contract’s runtime code**, sequentially combined.
    
    > **During contract creation, the EVM only executes the initialization code until it reaches the first STOP or RETURN instruction in the stack**. During this stage, the contract’s constructor() function is run, and the contract has an address. After this initialization code is run, only the runtime code remains on the stack. These opcodes are then copied into memory and returned to the EVM.

5. **Contract Address**
  - The contract address is determined by the sender's address and nonce. It becomes the unique identifier for the deployed contract on the Ethereum blockchain. 


# Subverting
