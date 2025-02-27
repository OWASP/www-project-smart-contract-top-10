## SC03:2025 - Logic Errors

### Description: 
Logic errors, also known as business logic vulnerabilities, are subtle flaws in smart contracts. They occur when the contract's code does not match its intended behavior. These errors can manifest in various forms, such as faulty math in reward distribution, improper token minting mechanisms, or incorrect calculations in lending and borrowing logic. Such vulnerabilities are elusive, hiding within the contract's logic and waiting to be discovered.

#### Examples of Logic Errors:
1. **Faulty Reward Distribution:** Miscalculations in dividing rewards among stakeholders, leading to unfair allocations.
2. **Improper Token Minting:** Unchecked or erroneous minting logic that allows infinite or unintended token generation.
3. **Lending Pool Imbalances:** Incorrect tracking of deposits and withdrawals, causing inconsistencies in pool reserves.

### Example (Vulnerable contract):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
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

    function mintReward(address to, uint256 rewardAmount) public {
        // Faulty minting logic: Reward amount not validated
        userBalances[to] += rewardAmount;
    }
}
```

### Impact:
- Logic errors can cause a smart contract to behave unexpectedly or even become entirely unusable. These errors can lead to:
  - **Loss of Funds:** Incorrect reward distribution or pool imbalances draining contract funds.
  - **Excessive Token Minting:** Inflating token supply, undermining trust and value.
  - **Operational Failures:** Contracts failing to perform their intended functions.
- These consequences can result in significant financial and operational losses for users and stakeholders.

### Remediation:
- Always validate your code by writing comprehensive test cases that cover all possible business logic scenarios.
- Conduct thorough code reviews and audits to identify and fix potential logic errors.
- Document the intended behavior of each function and module, and compare it to the actual implementation to ensure alignment.
- Implement guardrails, such as:
  - Safe math libraries to prevent calculation errors.
  - Proper checks and balances for token minting.
  - Auditable reward distribution algorithms.

### Example (Fixed version):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");

        // Correctly reducing the user's balance and updating the total lending pool
        userBalances[msg.sender] -= amount;
        totalLendingPool -= amount;

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        require(rewardAmount > 0, "Reward amount must be positive");

        // Safeguarded minting logic
        userBalances[to] += rewardAmount;
    }
}
```

### Examples of Smart Contracts That Fell Victim to Business Logic Attacks:
1. [Level Finance Hack](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO Hack](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)