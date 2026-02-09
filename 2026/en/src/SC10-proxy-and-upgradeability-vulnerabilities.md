## SC10:2026 - Proxy & Upgradeability Vulnerabilities

#### Description

Proxy and upgradeability vulnerabilities describe any situation where a smart contract uses an upgradeable architecture (proxy, beacon, or implementation-swapping pattern) and the upgrade path, initialization, or admin controls are misdesigned or misconfigured. Upgradeable contracts separate a **proxy** (which holds state and delegates calls) from an **implementation** (which contains logic). When upgradeability is improperly secured, attackers can hijack the proxy admin or upgrade role to deploy malicious implementations, re-initialize contracts to seize ownership, or bypass critical checks in initialization or migration steps.

This affects all contract types that use upgradeability: DeFi (lending, vaults, DEXes), NFTs (collections, marketplaces), DAOs (governance, treasuries), bridges (messengers, asset contracts), and L2/cross-chain systems. Common patterns include Transparent Proxy, UUPS (EIP-1822), Beacon Proxy, and custom router-implementation designs. On non-EVM chains, analogous upgrade mechanisms exist (e.g., Move modules, Solana program upgrades) with similar trust and initialization risks.

Few areas to focus on:

- **Upgrade and admin roles** (who can change the implementation, storage layout compatibility)
- **Initialization and re-initialization** (unprotected `initialize`, missing `initializer` guards, storage collisions)
- **Proxy delegation** (`delegatecall` context, `msg.sender`/`msg.value` propagation)
- **Storage layout** (slot collisions between proxy and implementation, append-only storage)
- **Timelocks and governance** (upgrade process, rollback capability)

Attackers exploit:

- **Unprotected upgrade functions** that allow any caller to point the proxy to a malicious implementation
- **Re-initialization** to reset ownership, configuration, or access controls
- **Initialization through delegatecall** where implementation can be initialized with attacker-controlled parameters
- **Storage collision** between proxy and implementation leading to overwrites

These issues often overlap with access control (SC01) but warrant separate attention due to the systemic impact of proxy and upgrade mechanisms.

### Example (Vulnerable Upgradeable Proxy Admin)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableProxyAdmin {
    address public admin;
    address public implementation;

    constructor(address _implementation) {
        // Critical: no way to set custom admin; implicitly trusts deployer logic
        admin = msg.sender;
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external {
        // Missing: access control (only admin) and sanity checks
        implementation = newImplementation;
    }
}
```

**Issues:**

- No access control on `upgrade`; any caller can change `implementation`.
- No checks on `newImplementation` (e.g., interface compatibility, non-zero address).

### Example (Safer Upgradeability Pattern)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract SafeProxyAdmin is Ownable {
    address public implementation;

    event Upgraded(address indexed newImplementation);

    error InvalidImplementation();

    constructor(address _implementation) {
        _setImplementation(_implementation);
        _transferOwnership(msg.sender);
    }

    function _setImplementation(address _implementation) internal {
        if (_implementation == address(0)) revert InvalidImplementation();
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external onlyOwner {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
}
```

**Security Improvements:**

- `upgrade` is restricted to the contract owner (which should itself be a robust governance or multisig).
- Implementation addresses are validated and upgrades are logged via events.

### Initialization & Re-Initialization Risks

Initialization functions (e.g., `initialize()`, `initializer` modifiers in OpenZeppelin) are critical in upgradeable patterns. Common pitfalls:

- **Unprotected initializers** that anyone can call.
- **Re-initialization** that can reset ownership, configuration, or state.
- Initialization logic that can be reached **through delegatecalls** from proxies in unintended ways.

Basic example:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableLogic {
    address public owner;

    // Missing initializer guard
    function initialize(address _owner) external {
        owner = _owner;
    }
}
```

If used behind a proxy without proper initialization control, attackers can call `initialize` via the proxy and set themselves as owner, taking over the protocol.

### 2025 Case Studies

- **Kinto Protocol (July 2025, $1.55M loss)**  
  Attackers exploited **uninitialized ERC1967 proxy contracts**. They detected freshly deployed proxy contracts that had not been properly initialized, then initialized them with malicious implementations containing dormant backdoors. Months later, the attacker activated the backdoor, upgraded the proxy to malicious code, and minted K tokens directly to drain $1.55M. The vulnerability: **unprotected initialization** allowing anyone to become the proxy admin.  

- **Uninitialized Proxy Campaign (2025, $10M+ across protocols)**  
  A broader campaign targeted uninitialized ERC1967 proxies across multiple EVM chains. Attackers used automated scanning to detect newly deployed proxies before legitimate developers could initialize them, then initialized with malicious implementations. The backdoors lay dormant for months, evading audits. When activated, attackers could upgrade proxies and drain funds.  
  - [https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses](https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses)
  - [https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6](https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6)

### Best Practices & Mitigations

- **Use well-established proxy patterns and libraries** (e.g., OpenZeppelin UUPS/transparent proxies) instead of bespoke designs.
- Protect upgrade and admin roles with **robust governance / multisigs**; never leave them on EOAs without strong operational controls.
- Apply **initializer guards**:
  - Use `initializer` and `reinitializer` modifiers correctly.
  - Lock implementation contracts once deployed to prevent direct initialization.
- Require **timelocks and multi-step processes** for upgrades:
  - Announce upgrade proposals.
  - Allow time for review/monitoring before execution.
- Maintain comprehensive **upgrade runbooks** and checklists, including:
  - Testing of migrations.
  - Verification of new implementation code and storage layout.
  - On-chain simulation of upgrade steps where possible.

