## SC06:2026 - Unchecked External Calls

#### Description

Unchecked external calls describe any situation where a smart contract invokes another contract or address (via `call`, `delegatecall`, `staticcall`, or high-level calls like `transfer`/`send`) **without fully accounting for** the callee’s behavior, return value, or reentrancy potential. The calling contract implicitly trusts that the callee will behave as expected—returning success, not re-entering, and not executing arbitrary logic. When that assumption is violated, the caller can be left in an inconsistent state or exploited.

This affects all contract types that perform external interactions: DeFi (token transfers, DEX swaps, vault deposits, flash loan callbacks), NFTs (transfers with hooks, marketplace payouts), DAOs (execution of proposal calldata), bridges (message relay, asset transfers), and composable protocols (arbitrary callbacks, ERC-777/ERC-721/ERC-1155 receiver hooks, ERC-4626 deposit/withdraw hooks). On non-EVM chains, analogous patterns exist (e.g., Move’s `vector::borrow`, Solana CPI) where cross-program invocations can re-enter or behave unexpectedly.

Few areas to focus on:

- **Token transfers** (ERC-20, ERC-721, ERC-1155) and non-standard return values or reverting behavior
- **Callback and hook interfaces** (ERC-777 `tokensReceived`, ERC-4626 `afterDeposit`/`beforeWithdraw`, `onFlashLoan`, `onERC721Received`)
- **Low-level calls** (`call`, `delegatecall`, `callcode`) and gas/storage implications
- **Composability flows** (vault calling strategy, strategy calling DEX) where reentrancy can cross multiple contracts

Attackers exploit:

- **Reentrancy** by implementing malicious logic in callbacks or in tokens that hook into transfers (see SC08)
- **Silent failures** when return values are ignored (e.g., non-returning ERC-20s) and state is left inconsistent
- **Unexpected code execution** when calling user-supplied or protocol-configurable addresses

Unchecked external calls are rarely the *sole* root cause but are a **critical enabler** for reentrancy (SC08), business logic exploits (SC02), and accounting inconsistencies.

### Example (Vulnerable Unchecked Call Pattern)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerablePayout {
    IToken public token;

    mapping(address => uint256) public rewards;

    constructor(IToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        require(amount > 0, "no rewards");

        // Vulnerable: does not check return value or reentrancy
        token.transfer(msg.sender, amount);

        // State update after external call
        rewards[msg.sender] = 0;
    }
}
```

**Issues:**

- The `transfer` call’s return value is ignored; if transfer fails, rewards remain non-zero but the user did not receive tokens.
- State is updated **after** the external call, opening reentrancy possibilities (if the token is malicious).

### Example (Hardened External Call Handling)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ISafeToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafePayout {
    ISafeToken public immutable token;
    mapping(address => uint256) public rewards;

    error NoRewards();
    error TransferFailed();

    constructor(ISafeToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        if (amount == 0) revert NoRewards();

        // Move state change *before* external call to mitigate reentrancy on this variable
        rewards[msg.sender] = 0;

        bool ok = token.transfer(msg.sender, amount);
        if (!ok) {
            // revert and restore state if needed (not shown here for brevity)
            revert TransferFailed();
        }
    }
}
```

**Security Improvements:**

- State is updated **before** the external call, limiting simple reentrancy on `rewards`.
- The token transfer’s return value is checked; failure results in a revert, preventing silent inconsistencies.

> Note: For full reentrancy protection, see SC08 and consider `ReentrancyGuard`, checks-effects-interactions, and pull-based patterns.

### 2025 Case Studies

- **GMX (July 2025, $42M loss)**  
  Unsafe external interactions and state updates after external calls allowed attackers to re-enter and manipulate accounting. The `executeDecreaseOrder` function transferred control to an attacker-supplied contract address during the refund process, enabling reentrancy. External call ordering, lack of proper checks, and reliance on assumptions about callee behavior amplified the impact.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)

- **Arcadia Finance (July 2025, $3.5M loss)**  
  The `SwapLogic._swapRouter()` and `RebalancerSpot` contracts allowed **arbitrary external calls** to user-supplied router addresses via `swapData` parameters, without validating the callee. The attacker registered a malicious contract as both the router and a whitelisted ArcadiaAccount, then used the router callback to spoof privileged execution contexts and invoke `setAssetManager()` / `flashAction()`. The protocol assumed the router would not have elevated permissions—an assumption not enforced in code. Unchecked callbacks and trust in external callee behavior enabled the drain.  
  - [https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d](https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d)
  - [https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025](https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025)

### Best Practices & Mitigations

- **Treat all external calls as untrusted**:
  - Even “standard” tokens or well-known protocols can be upgraded or replaced.
  - Assume they may re-enter or revert unexpectedly.
- Use the **checks-effects-interactions** pattern:
  - Validate pre-conditions.
  - Update internal state.
  - Only then perform external calls.
- Prefer **pull** over **push** for payments:
  - Allow users to withdraw rather than pushing funds to arbitrary addresses in loops.
- Check return values and handle all failure modes:
  - Use libraries like OpenZeppelin’s `SafeERC20` to wrap token operations.
- Be extremely careful with:
  - Low-level calls (`call`, `delegatecall`, `callcode`)
  - Arbitrary callbacks (e.g., hooks, onERC721Received, onFlashLoan callbacks)

