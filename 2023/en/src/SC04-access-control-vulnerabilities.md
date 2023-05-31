# Access Control Vulnerabilities

### Description
Access control vulnerabilities exist when a contract fails to properly restrict who can call certain functions. This can result in unauthorized function calls.

### Impact
If a contract function isn't protected adequately, unauthorized actors can manipulate the contract state, steal funds, or take other damaging actions.

### Steps to Fix
1. Use access control patterns such as Ownable or RBAC (Role-Based Access Control) in your contracts.
2. Regularly audit the contract for potential access control vulnerabilities.
3. Limit the capabilities of individual functions and roles within the contract.

### Example
The Parity Wallet vulnerability resulted from an unprotected function in a library contract, allowing an attacker to take ownership of the contract and self-destruct it, freezing over 500,000 Ether.
