## SC05:2025 - Reentrancy

### Description:
A reentrancy attack exploits the vulnerability in smart contracts when a function makes an external call to another contract before updating its own state. This allows the external contract, possibly malicious, to reenter the original function and repeat certain actions, like withdrawals, using the same state. Through such attacks, an attacker can possibly drain all the funds from a contract.

### Example (Vulnerable contract): 
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Vulnerability: Ether is sent before updating the user's balance, allowing reentrancy.
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // Update balance after sending Ether
        balances[msg.sender] = 0;
    }
}
```
### Impact:
- The most immediate and impactful consequence is the draining of funds. Attackers exploit vulnerabilities to withdraw more money than they are entitled to, potentially emptying the contract's balance completely.
- An attacker can trigger unauthorized function calls. This can lead to unintended actions being executed within the contract or related systems.

### Remediation:
- Always ensure that every state change happens before calling external contracts, i.e., update balances or code internally before calling external code.
- Use function modifiers that prevent reentrancy, like Open Zepplin’s Re-entrancy Guard.

### Example (Fixed version):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Fix: Update the user's balance before sending Ether
        balances[msg.sender] = 0;

        // Then send Ether
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### Examples of Smart Contracts that fell victim to Reentrancy Attacks:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)