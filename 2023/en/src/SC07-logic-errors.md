## Vulnerability: Logic Errors

### Description: 
Logic errors, also known as business logic vulnerabilities, are subtle flaws in smart contracts. They occur when the contract's code does not match its intended behavior. These errors are elusive, hiding within the contract's logic and waiting to be discovered.

### Example :
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LendingPlatform {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");
        
        // Faulty calculation: Incorrectly reducing the user's balance without updating the total lending pool
        userBalances[msg.sender] -= amount;
        
        // This should update the total lending pool, but it's omitted here.
        
        payable(msg.sender).transfer(amount);
    }
}
```
### Impact:
- Logic errors can cause a smart contract to behave unexpectedly or even become entirely unusable. These errors can result in the loss of funds, incorrect distribution of tokens, or other adverse outcomes, potentially leading to significant financial and operational consequences for users and stakeholders.
  
### Remediation:
- Always validate your code by writing comprehensive test cases that cover all the possible business logic.
- Conduct thorough code reviews and audits to identify and fix potential logic errors.
- Document the intended behavior of each function and module, and then compare it to the actual implementation to ensure alignment.

### Examples of Smart Contracts That Fell Victim to Business Logic Attacks:
1. [Level Finance Hack](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO Hack](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)
