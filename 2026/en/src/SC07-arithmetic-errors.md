## SC07:2026 - Arithmetic Errors (Rounding & Precision)

#### Description

Arithmetic errors (rounding and precision loss) describe any situation where a smart contract performs integer-based calculations that produce incorrect or exploitable results due to truncation, scaling, or unit conversion. Smart contracts are limited to integer arithmetic; any division, fixed-point scaling, or conversion between units can lose precision, introduce asymmetric rounding, or—when combined with unchecked blocks or non-EVM semantics—cause overflow/underflow (see SC09).

This affects all contract types that compute numeric values: DeFi (share minting/burning, LP tokens, interest accrual, swap outputs, AMM invariant updates), yield vaults and ERC-4626 (asset/share conversions), rebasing tokens, reward distribution, and NFT/token economics. On non-EVM chains (e.g., Move, Rust-based), integer semantics and available precision differ; similar risks apply wherever arithmetic drives economic outcomes.

Few areas to focus on:

- **Share and LP token calculations** (deposit/withdraw formulas, rounding direction)
- **Interest and reward accrual** (compounding, time-weighted averages)
- **Swap and AMM math** (constant product, concentrated liquidity, output calculations)
- **Fixed-point and scaling** (1e18, 1e8 conventions, cross-token conversions)
- **Rebasing and proportional distribution** (per-user vs. global accounting)

Attackers exploit:

- **Rounding bias** (e.g., rounding that favors depositor or protocol under adversarial sequences)
- **Repeated small gains** via flash loans (SC04) or high-frequency interactions
- **Edge cases** (zero total supply, first depositor, extreme ratios) where formulas break down
- **Precision loss** in multi-step computations that accumulates across operations

When combined with flash loans (SC04) or business logic flaws (SC02), arithmetic errors can be amplified into protocol-draining exploits.

### Example (Vulnerable Share Calculation)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableShares {
    uint256 public totalAssets;
    uint256 public totalShares;

    mapping(address => uint256) public balanceOf;

    function deposit(uint256 assets) external {
        require(assets > 0, "zero");

        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            // Rounds down and may favor the depositor under certain edge states
            shares = (assets * totalShares) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**Issues:**

- Rounding always truncates; under adversarial sequences of deposits/withdrawals, this can be gamed.
- No invariant tests to ensure that total share value remains consistent across edge cases.

### Example (More Robust Arithmetic & Invariant-Aware Design)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SaferShares {
    uint256 public totalAssets;
    uint256 public totalShares;

    mapping(address => uint256) public balanceOf;

    error ZeroAmount();
    error InvalidState();

    function deposit(uint256 assets) external {
        if (assets == 0) revert ZeroAmount();

        uint256 shares;
        if (totalShares == 0 || totalAssets == 0) {
            // Explicitly define initial conditions
            shares = assets;
        } else {
            // Use rounding strategy intentionally (up or down) and test it
            shares = (assets * totalShares + totalAssets - 1) / totalAssets; // round up
        }

        uint256 newTotalAssets = totalAssets + assets;
        if (newTotalAssets < totalAssets) revert InvalidState(); // overflow guard

        totalAssets = newTotalAssets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**Security Improvements:**

- Explicitly handles initial states and avoids division by zero.
- Uses a clearly documented rounding strategy (`round up` in this example).
- Adds overflow checks and custom errors for clarity.

> Real-world protocols should pair such logic with formal reasoning and invariants (e.g., through formal verification or property-based tests).

### 2025 Case Studies

- **zkLend (February 2025, $9.5M loss)**  
  A rounding error in the `mint()` function—integer division rounding down—caused a mismatch between recorded and actual values. A withdrawal that should have burned 1.5 tokens rounded down to 1.0, allowing attackers to artificially inflate the lending_accumulator via repeated deposits/withdrawals and drain ~$9.5M.  
  - [https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71](https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71)
  - [https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025](https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025)

- **Bunni (September 2025, $8.4M loss)**  
  Precision bugs in the withdrawal function's rounding logic. Developers assumed rounding down the idle balance would be "safe," but repeated operations created an exploitable loophole—the attacker decreased USDC balance by 85.7% while only burning 84.4% of liquidity, enabling disproportionate value extraction.  
  - [https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025](https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025)
  - [https://blog.bunni.xyz/posts/exploit-post-mortem/](https://blog.bunni.xyz/posts/exploit-post-mortem/)
  - [https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit](https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit)

### Best Practices & Mitigations

- **Use safe math patterns** (Solidity 0.8+ has built-in checks, but logic around them still matters).
- Clearly document and test your **rounding strategy**:
  - Decide whether rounding should favor the protocol or users.
  - Prove that repeated interactions cannot create “free value”.
- Rely on **well-reviewed math libraries** for complex operations:
  - Fixed-point math (e.g., 1e18 scaling)
  - High-precision exponentiation or logarithms
- Incorporate **invariant checks**:
  - E.g., `totalAssets` vs. sum of user balances, or share/value consistency after operations.
- Use **fuzz testing and differential testing** to discover edge cases around small/large values and repeated operations.

