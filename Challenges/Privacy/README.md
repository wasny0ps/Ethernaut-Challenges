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



# Subverting

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
