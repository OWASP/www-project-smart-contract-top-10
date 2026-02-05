## SC01:2026 - Access Control Vulnerabilities

#### Description

Improper access control describes any situation where a smart contract does not rigorously enforce *who* may invoke privileged behavior, *under which* conditions, and *with which* parameters. In modern DeFi systems this goes far beyond a single `onlyOwner` modifier. Governance contracts, multisigs, guardians, proxy admins, and cross‑chain routers all participate in enforcing who can mint or burn tokens, move reserves, reconfigure pools, pause or unpause core logic, or upgrade implementations. If any of these trust boundaries are weak or inconsistently applied, an attacker may be able to impersonate a privileged actor or cause the system to treat an untrusted address as if it were authorized.

Few areas to focus on: 
- **Ownership / admin controls** (e.g., `onlyOwner`, governor, multisig)
- **Upgrade and pause mechanisms** (proxy admins, guardians)
- **Fund movement and accounting** (mint/burn, pool reconfiguration, fee routing)
- **Cross-chain or cross-module trust boundaries** (bridges, vault routers, L2 messengers)

Attackers exploit:

- **Missing modifiers or role checks** on sensitive functions
- **Incorrect assumption of `msg.sender`** (e.g., via delegate calls or meta-transactions)
- **Unprotected initialization / re-initialization** of contracts or proxies
- **Privilege confusion** across modules (e.g., off-by-one checks, mistaken trusted addresses)

When combined with other issues (e.g., logic bugs, upgradeability flaws), access control failures can lead to full protocol compromise.

### Example (Vulnerable Contract)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LiquidityPoolVulnerable {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    // Anyone can set a new owner – critical access control bug
    function transferOwnership(address newOwner) external {
        owner = newOwner; // No access control
    }

    // Intended to be called only by the owner to rescue tokens
    function emergencyWithdraw(address to, uint256 amount) external {
        // Missing: require(msg.sender == owner)
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;
    }
}
```

**Issues:**

- `transferOwnership` is callable by anyone, allowing arbitrary takeover.
- `emergencyWithdraw` lacks any access control, effectively granting any caller the ability to drain the contract’s balance.

### Example (Fixed Version with Role-Based Access Control)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract LiquidityPoolSecure is AccessControl {
    bytes32 public constant GOVERNANCE_ROLE = keccak256("GOVERNANCE_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    mapping(address => uint256) public balances;

    event EmergencyWithdraw(address indexed to, uint256 amount, address indexed triggeredBy);

    constructor(address governance, address guardian) {
        _grantRole(DEFAULT_ADMIN_ROLE, governance);
        _grantRole(GOVERNANCE_ROLE, governance);
        _grantRole(GUARDIAN_ROLE, guardian);
    }

    function grantGovernance(address newGov) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(GOVERNANCE_ROLE, newGov);
    }

    function setGuardian(address newGuardian) external onlyRole(GOVERNANCE_ROLE) {
        _grantRole(GUARDIAN_ROLE, newGuardian);
    }

    // Only governance or designated guardians can trigger emergency withdrawals
    function emergencyWithdraw(address to, uint256 amount)
        external
        onlyRole(GUARDIAN_ROLE)
    {
        require(to != address(0), "invalid to");
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;

        emit EmergencyWithdraw(to, amount, msg.sender);
    }
}
```

**Security Improvements:**

- Explicit **RBAC**: `DEFAULT_ADMIN_ROLE`, `GOVERNANCE_ROLE`, and `GUARDIAN_ROLE`.
- Only trusted roles can reconfigure privileges or perform emergency withdrawals.
- Clear separation between configuration (governance) and emergency response (guardian).

### 2025 Case Studies

- **Balancer V2 (November 2025, ~$128M loss)**  
  A complex multi-chain pool ecosystem suffered from flawed access control in pool configuration and ownership assumptions. The `manageUserBalance` function had improper access controls—it checked `msg.sender` against a user-provided `op.sender` value, which attackers could set to match `msg.sender` and bypass protections, allowing them to masquerade as pool controllers and execute unauthorized WITHDRAW_INTERNAL operations. This was chained with a rounding error in `_upscaleArray` to drain liquidity.  
  Key lessons:
  - Critical pool operations must be **guarded by explicit role checks** and on-chain governance.
  - Any cross-chain or cross-module "owner" concept must be **verified on-chain**, not assumed from message origin.  
  - [https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit)
  - [https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/)
  - [https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025](https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025)

- **Zoth (March 2025, $8.4M loss)**  
  Improper privilege checks around core accounting and administrative functions allowed attackers to perform unauthorized fund movements. The attacker compromised Zoth's deployer wallet (single EOA controlling admin) and performed a malicious upgrade to the USD0PPSubVaultUpgradeable proxy, deploying a malicious implementation to withdraw $8.4M. The protocol relied on brittle assumptions—a single private key controlling critical admin functions.  
  Key lessons:
  - Avoid **implicit trust** in addresses (e.g., "deployer is trusted forever").
  - Use **role-based access control (RBAC)** with clear separation between operational, emergency, and upgrade roles.  
  - [https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b](https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b)
  - [https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025](https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025)

- **Cork Protocol (May 2025, $11–12M loss)**  
  The Uniswap V4 hook callbacks (e.g., `beforeSwap`) lacked proper access control—they did not validate that the caller was the trusted PoolManager. The `beforeSwap` function had no `onlyPoolManager` modifier. Attackers called the hook directly with arbitrary parameters, fooling the protocol into crediting them with derivative tokens. The root cause was **missing caller validation** on hook entry points.  
  Key lessons:
  - Hook and callback entry points must **validate the caller** (e.g., onlyPoolManager) explicitly on-chain.  
  - [https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/](https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/)
  - [https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit](https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit)
  - [https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025)

### Best Practices & Mitigations

Robust access control starts with using battle‑tested primitives such as OpenZeppelin’s `Ownable` and `AccessControl` rather than bespoke role systems. Privileged roles should be few, clearly documented, and ideally held by well‑secured multisigs or governance modules instead of EOAs. Initialization routines for upgradeable contracts must be locked after first use, with `initializer`/`reinitializer` guards and explicit versioning to prevent re‑initialization attacks. Upgrade paths for proxies and core components should be tightly controlled and observable, with events emitted for every privilege change or upgrade so that off‑chain monitoring can quickly detect abuse. Finally, access control policies should be encoded in tests, fuzzing properties, and, where possible, formal specifications, verifying properties such as “no unprivileged address can ever drain funds or seize admin control.”

