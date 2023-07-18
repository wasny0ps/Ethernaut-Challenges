<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel16.svg">

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


The **Preservation** contract is a smart contract that allows users to set the time for **two different time zones using delegate calls to library contracts**. The contract is designed to be **upgradable**, meaning that the logic for setting the time is separated into external library contracts, **which can be swapped without modifying the main contract**. The `timeZone1Library` and `timeZone2Library` variables store the addresses of two public library contracts that handle the logic for setting the time in different time zones. And there is **storedTime** variable which stores the timestamp set by the library contracts.



# Subverting
