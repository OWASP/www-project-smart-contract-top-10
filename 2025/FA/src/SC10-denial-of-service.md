## SC10:2025 - Denial Of Service

### Description:
A Denial of Service (DoS) attack in Solidity involves exploiting vulnerabilities to exhaust resources like gas, CPU cycles, or storage, making a smart contract unusable. Common types include gas exhaustion attacks, where malicious actors create transactions requiring excessive gas, reentrancy attacks that exploit contract call sequences to access unauthorized funds, and block gas limit attacks that consume block gas, hindering legitimate transactions.

### Example (Vulnerable contract):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        //If the current king has a malicious fallback function that reverts, it will prevent the new king from claiming the throne, causing a Denial of Service.
        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```
### Impact:
- A successful DoS attack can render the smart contract unresponsive, preventing users from interacting with it as intended. This can disrupt critical operations and services relying on the contract.
-  DoS attacks can lead to financial losses, especially in decentralized applications (dApps) where smart contracts manage funds or assets.
- A DoS attack can tarnish the reputation of the smart contract and its associated platform. Users may lose trust in the platform's security and reliability, leading to a loss of users and business opportunities.
  
### Remediation:
- Ensure smart contracts can handle consistent failures, such as asynchronous processing of potentially failing external calls, to maintain contract integrity and prevent unexpected behavior.
- Be cautious when using `call` for external calls, loops, and traversals to avoid excessive gas consumption, which could lead to failed transactions or unexpected costs.
- Avoid over-authorizing a single role in contract permissions. Instead, divide permissions reasonably and use multi-signature wallet management for roles with critical permissions to prevent permission loss due to private key compromise.

### Example (Fixed version):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    // Use a safer approach to transfer funds, like transfer, which has a fixed gas stipend.
    // This avoids using call and prevents issues with malicious fallback functions.
    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        address previousKing = king;
        uint256 previousBalance = balance;

        // Update the state before transferring Ether to prevent reentrancy issues.
        king = msg.sender;
        balance = msg.value;

        // Use transfer instead of call to ensure the transaction doesn't fail due to a malicious fallback.
        payable(previousKing).transfer(previousBalance);
    }
}
```