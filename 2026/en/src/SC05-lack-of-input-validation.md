## SC05:2026 - Lack of Input Validation

#### Description

Lack of input validation describes any situation where a smart contract processes external data—function parameters, calldata, cross-chain messages, or signed payloads—**without rigorously enforcing** that the data is well-formed, within expected bounds, and authorized for the intended operation. Contracts that assume inputs are benign leave themselves open to malformed or adversarial data that pushes the system into unsafe states, corrupts accounting, or bypasses intended checks.

This applies across all contract types: DeFi (fee bps, slippage, amounts, addresses), NFTs (token IDs, metadata, royalty config), DAOs (proposal payloads, voting parameters), bridges (message payloads, destination chains), and generic composable contracts that accept arbitrary calldata or relayed calls. On non-EVM chains, the same principle holds: untrusted inputs from users, other contracts, or cross-chain channels must be validated before use.

Few areas to focus on:

- **Numeric parameters** (amounts, fees, rates, slippage, collateral factors) and safe bounds
- **Addresses** (zero address, contract vs. EOA assumptions, delegated or proxy addresses)
- **Off-chain and signed data** (signatures, expiry, nonce replay)
- **Cross-chain and bridge payloads** (message format, chain ID, sender verification)
- **Admin and governance inputs** (configuration values, upgrade parameters)—often treated as trusted but can be misconfigured or exploited

Attackers exploit:

- **Out-of-bounds values** (e.g., fee > 100%, zero amounts, max uint) that break invariants
- **Malformed addresses or payloads** that bypass allowlists or cause unexpected behavior
- **Replay and ordering attacks** when nonce/expiry/chain ID are not validated
- **Composability edge cases** when contracts assume caller format or trust relayed data

In 2025, input validation issues often appeared as a *contributing factor*, e.g., failure to enforce safe ranges on parameters controlling liquidity or interest computations.

### Example (Vulnerable Parameter Handling)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableConfig {
    uint256 public feeBps;      // basis points 0–10_000
    uint256 public maxDeposit;  // upper bound for deposits

    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        // Missing: access control and bounds checks
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }
}
```

**Issues:**

- No access control: anyone can call `setConfig`.
- No validation of `_feeBps` or `_maxDeposit`:
  - `feeBps` could exceed 100%, breaking fee logic.
  - `maxDeposit` could be set to an unsafe or zero value, disrupting the protocol.

### Example (Fixed Version with Strong Validation)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeConfig {
    uint256 public feeBps;      // 0–1_000 (max 10% fee)
    uint256 public maxDeposit;  // upper bound for deposits

    address public immutable owner;

    error NotOwner();
    error InvalidFee();
    error InvalidMaxDeposit();

    constructor(uint256 initialFeeBps, uint256 initialMaxDeposit) {
        owner = msg.sender;
        _setConfig(initialFeeBps, initialMaxDeposit);
    }

    function _setConfig(uint256 _feeBps, uint256 _maxDeposit) internal {
        if (_feeBps > 1_000) revert InvalidFee();
        if (_maxDeposit == 0) revert InvalidMaxDeposit();
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        if (msg.sender != owner) revert NotOwner();
        _setConfig(_feeBps, _maxDeposit);
    }
}
```

**Security Improvements:**

- Validates that fee is bounded within a documented, safe range.
- Requires `maxDeposit` to be non-zero, preventing misconfiguration.
- Restricts configuration changes to the contract owner (see SC01 for more advanced RBAC).

### 2025 Case Studies

- **Cetus (May 2025, $223M loss)**  
  The primary root cause was a flawed overflow check in `checked_shlw` (see SC09). However, **insufficient input validation** was a contributing factor—the protocol allowed extreme liquidity parameters (e.g., ~2^113) without bounds checks. When combined with the flawed arithmetic, these unvalidated inputs produced dangerous edge cases leading to pool drains.  
  - [https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
  - [https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)
  - [https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025)

- **Ionic Money (February 2025, ~$6.9M loss)**  
  Attackers used **social engineering** to convince the protocol to list a counterfeit LBTC token. Once listed, the protocol accepted it as collateral without validating token authenticity on-chain (e.g., verification that listed collateral contracts are legitimate). Attackers minted 250 fake LBTC and used it to borrow ~$8.6M. *Note: The root cause was partly off-chain (governance/listing process); the on-chain vulnerability was insufficient validation that collateral tokens are genuine before trusting whitelisted addresses.*  
  - [https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025](https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025)
  - [https://rekt.news/ionic-money-rekt](https://rekt.news/ionic-money-rekt)

### Best Practices & Mitigations

- **Validate all external inputs**, including:
  - Function parameters (amounts, addresses, configuration values)
  - Off-chain-signed data and calldata payloads
  - Cross-chain messages and bridge payloads
- Enforce **tight invariants**:
  - Ranges for fees, interest rates, leverage, and collateral factors.
  - Non-zero requirements for key addresses and limits.
- Use **custom errors** and explicit checks to keep validation clear and gas-efficient.
- Treat **admin and governance inputs** as untrusted until validated—misconfiguration can be as damaging as explicit exploits.
- Include **negative tests** for invalid inputs (fuzzing, property tests) to ensure unexpected values are rejected.

