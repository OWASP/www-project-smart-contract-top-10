## SC08:2026 - Reentrancy Attacks

#### Description

Reentrancy describes any situation where a smart contract performs an external call (to another contract or address), and the callee can **call back into** the original contract before the first invocation has completed and state has been fully updated. If the caller is not designed to be reentrancy-safe, the callback can observe stale state and exploit it—e.g., withdrawing more than the caller’s balance, double-counting rewards, or manipulating accounting across complex multi-step flows.

This affects all contract types that perform external calls: DeFi (token transfers, DEX swaps, vault deposits/withdrawals, flash loan callbacks), NFTs (transfers with ERC-721/ERC-1155 receiver hooks, marketplace payouts), DAOs (proposal execution that invokes external contracts), bridges (message relay, asset transfers), and composable protocols (ERC-777 hooks, ERC-4626 deposit/withdraw hooks). Reentrancy can be **single-function** (same function called recursively), **cross-function** (callback into a different function), or **cross-contract** (callback traverses multiple contracts). On non-EVM chains, analogous patterns exist wherever cross-program invocations can recurse.

Few areas to focus on:

- **Classic withdraw-before-update** (state change after external call)
- **Callback and hook interfaces** (ERC-777, ERC-721/1155 receivers, ERC-4626, flash loan callbacks)
- **Cross-function reentrancy** (read-your-writes assumptions between functions)
- **Read-only reentrancy** (view functions or oracles reading state during a callback)
- **Cross-contract and multi-module** reentrancy (vault → strategy → DEX flows)

Attackers exploit:

- **Malicious tokens or receivers** that re-enter on transfer/hook callbacks
- **Flash loan callbacks** that execute attacker logic before repayment
- **Complex call graphs** where state is inconsistent mid-transaction across modules

### Example (Vulnerable Reentrancy Pattern)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerableVault {
    IToken public immutable token;
    mapping(address => uint256) public balances;

    constructor(IToken _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // Assume token already transferred in for brevity
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "insufficient");

        // External call before state update – reentrancy window
        token.transfer(msg.sender, amount);

        balances[msg.sender] -= amount;
    }
}
```

**Issues:**

- An attacker using a malicious token / proxy can re-enter `withdraw` from within the `transfer` call, repeatedly withdrawing based on the unchanged `balances[msg.sender]`.

### Example (Reentrancy-Safe Withdraw)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface ITokenSafe {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafeVault is ReentrancyGuard {
    ITokenSafe public immutable token;
    mapping(address => uint256) public balances;

    error InsufficientBalance();
    error TransferFailed();

    constructor(ITokenSafe _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // Assume token already transferred in for brevity
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external nonReentrant {
        uint256 bal = balances[msg.sender];
        if (bal < amount) revert InsufficientBalance();

        // Effects first
        balances[msg.sender] = bal - amount;

        // Then interaction
        bool ok = token.transfer(msg.sender, amount);
        if (!ok) revert TransferFailed();
    }
}
```

**Security Improvements:**

- Uses `nonReentrant` modifier from OpenZeppelin’s `ReentrancyGuard`.
- Applies **checks-effects-interactions** pattern to minimize reentrancy windows.
- Checks transfer return value and reverts on failure.

### 2025 Case Study

- **GMX (July 2025, $42M loss)**  
  GMX V1 contracts were exploited via a classic yet sophisticated reentrancy vector in `executeDecreaseOrder`. The function accepted the attacker's smart contract address as a parameter; when it transferred control to that address during the refund process, the attacker re-entered and manipulated global average short prices, AUM, and GLP valuations. State updates after external calls and lack of reentrancy guards enabled the drain. The vulnerability was introduced in 2022 as an unaudited patch.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### Best Practices & Mitigations

- Use **`ReentrancyGuard`** or similar reentrancy locks on stateful functions that:
  - Modify balances or internal accounting.
  - Perform external calls that could re-enter the contract.
- Follow **checks-effects-interactions**:
  - Check preconditions.
  - Apply all state changes.
  - Only then call external contracts.
- Treat **ERC-777 hooks, ERC-4626 hooks, and other callbacks** as reentrancy vectors.
- Carefully review:
  - Cross-function interactions (e.g., `deposit` calling `withdraw` internally).
  - Multi-contract systems where reentrancy may occur across modules, not just within a single contract.
- Include reentrancy-focused **fuzzing and unit tests**, especially where external calls are involved.

