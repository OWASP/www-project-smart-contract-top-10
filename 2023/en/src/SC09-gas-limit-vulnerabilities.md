## Vulnerability: Gas Limit Vulnerabilities

### Description: 
In Ethereum and other blockchain platforms, every operation performed by a smart contract consumes a certain amount of gas, which is a unit of computational effort. The block gas limit is the maximum amount of gas that can be used in a single block. If a function in a smart contract requires more gas than the block gas limit to complete its execution, the transaction will fail. This type of vulnerability is particularly common in loops that iterate over dynamic data structures, such as arrays or lists, where the number of iterations is not fixed and can grow arbitrarily large.

### Example :
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TokenTransfer {
    mapping(address => uint256) public balances;

    function transfer(address _to, uint256 _amount) public {
        require(balances[msg.sender] >= _amount, "Insufficient balance");
        
        for (uint256 i = 0; i < _amount; i++) {  // The loop iterates _amount times, which can be very inefficient and can potentially exceed the block gas limit if _amount is too large.
            balances[msg.sender]--; 
            balances[_to]++; 
        }
    }
}
```
### Impact:
- Functions that are susceptible to gas limit issues can become unexecutable, resulting in locked funds or a frozen contract state. When these functions fail to complete due to exceeding the gas limit, any funds associated with the transaction remain inaccessible, effectively locking them within the contract.  

### Remediation:
- The functions should validate that the users can’t control the variable length used inside the loop to traverse a large amount of data. If it can’t be omitted, then there should be a limit on the length as per the code logic.
- Whenever loops are used in Solidity, the developers should pay special attention to the actions happening inside the loop to make sure that the transaction does not consume excessive gas and does not go over the gas limit.

