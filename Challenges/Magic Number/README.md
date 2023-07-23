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

In this challenge, we are facing a basic smart contract which saves the solver's address with the **setSolver()** function. Keep in mind that this challenge requires some assembly programming skills to deploy the solver contract. 


# Contract Creation

<p align="center"><img src="https://miro.medium.com/v2/resize:fit:1100/format:webp/1*5Wrb7z3W6AMtjH6IKJYowg.jpeg"></p>

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

In this section, we need to write two sets of bytecodes:

- `Initialization Opcodes`: Which EVM uses to create the contract by replicating the runtime opcodes and returning them to EVM to save in storage.
- `Runtime Opcodes`: This is the actual code run after the contract creation. In other words, this contains the logic of the contract.

## Runtime Opcodes

Thankfully, there's a limited size of 10 opcodes and 10 bytes since each opcode is 1 byte. That's why, our solver should be of at most 10 bytes and it should return 42 (`0x2a`). Our helpful opcode is `RETURN`. But, RETURN takes two arguments:
- **p** : The location of value in memory.
- **s** : The size of this value to be returned.

That means the 0x2a needs to be stored in memory first which `MSTORE(p,v)` facilitates. But MSTORE itself takes two arguments:
- **p** : The location of value in stack.
- **v** : Value's size.

So, we need push the value and size params into stack first using `PUSH1` opcode. 

Let's look at closer the opcodes to be used:

```assembly
0x60 - PUSH1 --> PUSH(0x2a) --> 0x602a // Pushing 2a or 42
0x60 - PUSH1 --> PUSH(0x80) --> 0x6080 // Pushing an arbitrary selected memory slot 80
0x52 - MSTORE --> MSTORE --> 0x52      //Store value p=0x2a at position v=0x80 in memory
```

Then, you return this the 0x42 value:

```assembly
0x60 - PUSH1 --> PUSH(0x20) --> 0x6020 // Size of value is 32 bytes
0x60 - PUSH1 --> PUSH(0x80) --> 0x6080 // Value was stored in slot 0x80
0xf3 - RETURN --> RETURN --> 0xf3      // Return value at p=0x80 slot and of size s=0x20
```
The resulting opcode sequence should be `604260805260206080f3`. This runtime opcode is exactly 10 opcodes and 10 bytes long. In this way, it fits the challenge's requirements.

## Initialization Opcodes

Time to focus on initialization opcodes. These opcodes responsible for loading runtime opcodes in memory and returning it to the EVM. To copy code, we need to use the `CODECOPY(t, f, s)` opcode which takes 3 arguments:

- **t** : The destination offset where the code will be in memory. Let's save this to 0x00 offset.
- **f** : This is the current position of the runtime opcode which is not known as of now.
- **s** : This is the size of the runtime code in bytes. Remember that `602a60805260206080f3` is 10 bytes long.

At first, copy the runtime opcodes into memory. Add a placeholder for `f`, as it is currently unknown:

```assembly
0x60 - PUSH1 --> PUSH(0x0a) --> 0x600a // s:0x0a or 10 bytes
0x60 - PUSH1 --> PUSH(0x??) --> 0x60?? // f:Current position of runtime opcodes but this is not known yet
0x60 - PUSH1 --> PUSH(0x00) --> 0x6000 // t:0x00 or destination memory slot 0
0x39 - CODECOPY --> CODECOPY --> 0x39  // Calling the CODECOPY with all the arguments
```
Then, it should return the runtime opcode to the EVM:

```assembly
0x60 - PUSH1 --> PUSH(0x0a) --> 0x600a // Size of opcode is 10 bytes
0x60 - PUSH1 --> PUSH(0x00) --> 0x6000 // Value was stored in slot 0x00
0xf3 - RETURN --> RETURN --> 0xf3      // Return value at p=0x00 slot and of size s=0x0a
```

The bytecode for the initialization opcode will become `600a60__600039600a6000f3` which is 12 bytes in total. This means the missing value for the starting position for the runtime opcode **f will be index 12 or 0x0c**, making our final bytecode `600a600c600039600a6000f3`.

In this turn, we can combine them to get the final bytecode which can be used to deploy the contract. `602a60805260206080f3` + `600a600c600039600a6000f3` = `600a600c600039600a6000f3602a60505260206050f3`.

Before the coffee break, we must do that create this solver contract with this bytecode in web3. See in [etherscan.](https://sepolia.etherscan.io/tx/0x7c5adb520e3d989447967562d7e611f8c65c9efb410982e8bf7e462a03d79ce2)

```shell
var bytecode = "600a600c600039600a6000f3602a60505260206050f3"
var txn = web3.eth.sendTransaction({from: player, data: bytecode})
```

Here is the solver contract's address `0xa1fdcbeafa437e6d16bbd99381112cc543e8a09b`. Finally, we must confirm this address to the challenge's contract. [See in etherscan.](https://sepolia.etherscan.io/tx/0xaea0d4f5a08a8fdf1f4191d1a27e4dddf976e3a9539a219b78b1d4312a1e00c4)

```shell
await contract.setSolver("0xa1fdcbeafa437e6d16bbd99381112cc543e8a09b")
await contract.solver()
'0xa1FDcBEaFa437e6d16BbD99381112Cc543e8A09B'
```

And we solved this!

Ethernaut's message: 

> Congratulations! If you solved this level, consider yourself a Master of the Universe. Go ahead and pierce a random object in the room with your Magnum look. Now, try to move it from afar; Your telekinesis habilities might have just started working.

**_by wasny0ps_**
