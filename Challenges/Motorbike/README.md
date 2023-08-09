<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel25.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

Challenge's message:

> Ethernaut's motorbike has a brand new upgradeable engine design. Would you be able to selfdestruct its engine and make the motorbike unusable ?


This set of Solidity contracts demonstrates a simple upgradeability pattern using EIP-1967, which enables you to upgrade the logic of a smart contract while maintaining its storage and state. EIP-1967 introduces a proxy contract that delegates calls to an implementation contract. This allows you to upgrade the implementation without changing the proxy contract's address or the storage layout.


The `Motorbike` contract is a proxy contract that delegates function calls to its implementation. Here's how it works:

- **Initialization**: The constructor of `Motorbike` takes an address `_logic` as an argument, which represents the initial implementation contract. It checks if `_logic` is a valid contract address using the `Address.isContract()` function from OpenZeppelin. Then, it sets the `_IMPLEMENTATION_SLOT` to the `_logic` address. After that, it attempts to delegate call the `initialize()` function of the `_logic` contract.

 - **Delegation**: The `fallback` function of the `Motorbike` contract is triggered when an unknown function is called on the proxy contract. It delegates the call to the current implementation by using the address stored in the `_IMPLEMENTATION_SLOT`.


The `Engine` contract is the implementation contract meant to be used with the `Motorbike` proxy. It inherits from `Initializable`, which is a part of the OpenZeppelin library.

- **Initialization**: The `initialize()` function initializes the contract state. It sets the `horsePower` to 1000 and the `upgrader` to the sender's address.

- **Upgrade and Call**: The `upgradeToAndCall()` function is used to upgrade the implementation of the proxy and execute a function on the new implementation. This function requires the sender to have the `upgrader` role. It calls `_authorizeUpgrade()` to ensure the sender is authorized. Then, it upgrades the implementation using `_upgradeToAndCall()`. If `data` is provided, it delegates calls to the new implementation with the provided data.

- **Authorization Check**: The `_authorizeUpgrade()` function restricts the upgrade functionality to the `upgrader` role. If the sender is not the `upgrader`, the upgrade is denied.

- **Implementation Upgrade**: The `_upgradeToAndCall()` function performs the actual implementation upgrade. It sets the new implementation address in the `_IMPLEMENTATION_SLOT` and delegates calls to the new implementation. If `data` is provided, it's used to call a function on the new implementation.

- **Implementation Storage**: The `_setImplementation()` function is responsible for storing the new implementation address in the `_IMPLEMENTATION_SLOT`.


This set of contracts implements a basic upgradeability pattern using EIP-1967, where a proxy contract (`Motorbike`) delegates calls to an implementation contract (`Engine`). The `Engine` contract can be upgraded by an authorized upgrader using the `upgradeToAndCall()` function. This enables the contract's logic to be upgraded while retaining its storage and state.

# Subverting


```solidity

```


Ethernaut's message:

> The advantage of following an UUPS pattern is to have very minimal proxy to be deployed. The proxy acts as storage layer so any state modification in the implementation contract normally doesn't produce side effects to systems using it, since only the logic is used through delegatecalls.
This doesn't mean that you shouldn't watch out for vulnerabilities that can be exploited if we leave an implementation contract uninitialized.
This was a slightly simplified version of what has really been discovered after months of the release of UUPS pattern.
Takeways: never leave implementation contracts uninitialized ;)
