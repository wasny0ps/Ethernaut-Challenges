## Exploring

Look at the ```abi``` of target contract.
```shell
await contract.abi
Array(11) [ {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, {…}, … ]
​
0: Object { inputs: (1) […], stateMutability: "nonpayable", type: "constructor", … }
​
1: Object { name: "authenticate", stateMutability: "nonpayable", type: "function", … }
​
2: Object { name: "getCleared", stateMutability: "view", type: "function", … }
​
3: Object { name: "info", stateMutability: "pure", type: "function", … }
​
4: Object { name: "info1", stateMutability: "pure", type: "function", … }
​
5: Object { name: "info2", stateMutability: "pure", type: "function", … }
​
6: Object { name: "info42", stateMutability: "pure", type: "function", … }
​
7: Object { name: "infoNum", stateMutability: "view", type: "function", … }
​
8: Object { name: "method7123949", stateMutability: "pure", type: "function", … }
​
9: Object { name: "password", stateMutability: "view", type: "function", … }
​
10: Object { name: "theMethodName", stateMutability: "view", type: "function", … }
​
length: 11
​
<prototype>: Array []

```
Call info() function.
```shell
await contract.info()
"You will find what you need in info1()."

await contract.info1()
"Try info2(), but with \"hello\" as a parameter."

await contract.info2("hello")
"The property infoNum holds the number of the next info method to call."

await contract.infoNum()
Object { negative: 0, words: (2) […], length: 1, red: null }
​
length: 1
​
negative: 0
​
red: null
​
words: Array [ 42, <1 empty slot> ]
​​
0: 42
​​
length: 2
​​
<prototype>: Array []
​
<prototype>: Object { _init: _init(t, e, r), _initNumber: _initNumber(t, e, r), _initArray: _initArray(t, e, r), … }
```
As you can see, the next info method is **info42()**.
```
await contract.info42()
"theMethodName is the name of the next method."
```
```shell
await contract.theMethodName()
"The method name is method7123949."

await contract.method7123949()
"If you know the password, submit it to authenticate()."
```
```shell
await contract.password()
"ethernaut0"

await contract.authenticate("ethernaut0")
⛏️ Sent transaction ⛏ https://rinkeby.etherscan.io/tx/0xb0b695faa36b7ed8b74633c14ccd4a09c27b50d66219d9f43e1f292b3e108ab3
```
It's the transaction hash.
> 0xb0b695faa36b7ed8b74633c14ccd4a09c27b50d66219d9f43e1f292b3e108ab3

Submit instance.
> Congratulations! You have completed the tutorial. Have a look at the Solidity code for the contract you just interacted with below.

You are now ready to complete all the levels of the game, and as of now, you're on your own.

Godspeed!!

**_by wasny0ps_**
