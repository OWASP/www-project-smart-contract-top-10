## SC01:2025 - Improper Access Control 

### Description:
An access control vulnerability is a security flaw that allows unauthorized users to access or modify the contract's data or functions. These vulnerabilities arise when the contract's code fails to adequately restrict access based on user permission levels. Access control in smart contracts can relate to governance and critical logic, such as minting tokens, voting on proposals, withdrawing funds, pausing and upgrading the contracts, and changing ownership.

### Example (Vulnerable contract):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_AccessControl {
    mapping(address => uint256) public balances;

    // Burn function with no access control
    function burn(address account, uint256 amount) public {
        _burn(account, amount);
    }
}
```
### Impact:
- Attackers can gain unauthorized access to critical functions and data within the contract, compromising its integrity and security.
- Vulnerabilities can lead to the theft of funds or assets controlled by the contract, causing significant financial damage to users and stakeholders.

### Remediation:
- Ensure initialization functions can only be called once and exclusively by authorized entities.
- Use established access control patterns like Ownable or RBAC (Role-Based Access Control) in your contracts to manage permissions and ensure that only authorized users can access certain functions. This can be done by adding appropriate access control modifiers, such as `onlyOwner` or custom roles to sensitive functions.

### Example (Fixed version):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Import the Ownable contract from OpenZeppelin to manage ownership
import "@openzeppelin/contracts/access/Ownable.sol";

contract Solidity_AccessControl is Ownable {
    mapping(address => uint256) public balances;

    // Burn function with proper access control, only accessible by the contract owner
    function burn(address account, uint256 amount) public onlyOwner {
        _burn(account, amount);
    }
}
```

### Examples of Smart Contracts That Fell Victim to Improper Access Control Attacks:
1. [HospoWise Hack](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT Hack](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)