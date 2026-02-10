## SC02:2026 - Business Logic Vulnerabilities

#### Description

Business logic vulnerabilities describe any situation where a smart contract’s *intended* economic or functional behavior can be subverted even though individual low-level checks (e.g., type safety, reentrancy guards, access control) are correct. These are **design flaws** in how the system’s rules, incentives, state transitions, and invariants are modeled on-chain. Unlike low-level bugs (overflow, reentrancy), business logic flaws arise when the rules themselves are unsafe—the code “does what it says”, but what it says permits exploitable outcomes.

This applies across all smart contract domains: DeFi (lending, AMMs, vaults, yield strategies), NFTs (minting logic, royalties, marketplace mechanics), DAOs (voting, delegation, proposal execution), bridges (burn/mint asymmetry, liquidity rules), gaming (reward distribution, fairness), and cross-chain/L2 systems where multi-hop state transitions create emergent vulnerabilities.

Few areas to focus on:

- **Invariant violations** across modules (e.g., vault ↔ strategy ↔ gauge, collateral ↔ debt, supply ↔ backing)
- **Reward and fee logic** (double-counting, under/over-accrual, wrong beneficiary)
- **Eligibility and limit checks** that are bypassable or inconsistently enforced (borrowing caps, mint limits, liquidation thresholds)
- **Path-dependent state machines** that allow manipulative action sequences to reach impossible or inconsistent states
- **Cross-module and cross-chain assumptions** (e.g., bridge liquidity, L2 finality, message ordering)

Attackers exploit:

- **Arbitrage between inconsistent accounting** (vault vs. strategy, internal vs. external balance)
- **Order-of-operations edge cases** (deposit/withdraw/claim sequences that break invariants)
- **Eligibility bypasses** (e.g., claiming rewards without stake, liquidating healthy positions)
- **Parameter manipulation** (interest curves, fees, collateral factors) that create economically irrational states

### Example (Vulnerable Lending Logic)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableLending {
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;
    uint256 public collateralFactorBps = 7500; // 75%

    function depositCollateral() external payable {
        collateral[msg.sender] += msg.value;
    }

    // Vulnerable: calculates borrow capacity using the *new* amount, not total
    function borrow(uint256 amount) external {
        uint256 allowed = (amount * collateralFactorBps) / 10_000;
        require(allowed >= amount, "not enough collateral"); // meaningless check

        debt[msg.sender] += amount;
        // send tokens from pool (omitted)
    }
}
```

**Issues:**

- `allowed` is computed from the *requested borrow amount*, not the user’s collateral balance.
- The check `allowed >= amount` always holds for `collateralFactorBps >= 10_000`, and is otherwise simply a tautology when misused like this, failing to enforce any economic invariant.

### Example (Fixed: Invariant-Based Borrow Logic)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IPriceOracle {
    function getCollateralPrice() external view returns (uint256); // 1e18
}

contract SaferLending {
    mapping(address => uint256) public collateral;
    mapping(address => uint256) public debt;
    uint256 public collateralFactorBps = 7500; // 75%
    IPriceOracle public oracle;

    constructor(IPriceOracle _oracle) {
        oracle = _oracle;
    }

    function depositCollateral() external payable {
        require(msg.value > 0, "no collateral");
        collateral[msg.sender] += msg.value;
    }

    function _maxBorrow(address user) internal view returns (uint256) {
        uint256 price = oracle.getCollateralPrice(); // e.g. ETH price in USD 1e18
        uint256 collateralUsd = (collateral[user] * price) / 1e18;
        return (collateralUsd * collateralFactorBps) / 10_000;
    }

    function borrow(uint256 amountUsd) external {
        uint256 maxBorrowUsd = _maxBorrow(msg.sender);
        require(debt[msg.sender] + amountUsd <= maxBorrowUsd, "exceeds limit");

        debt[msg.sender] += amountUsd;
        // transfer stablecoin from pool (omitted)
    }
}
```

**Security Improvements:**

- Borrow limits derive from **total collateral**, not just requested amounts.
- Economic invariant: `debt[user] <= maxBorrow(user)` is enforced on every borrow.
- Price oracle is explicitly integrated (and can be hardened per SC03).

### 2025 Case Studies

- **Abracadabra (March 2025, $12.9M loss)**  
  Flawed collateral accounting in GMX V2 CauldronV4 contracts. Attackers used a three-stage method: (1) made a deposit into GMX designed to fail, leaving tokens stuck in OrderAgent; (2) self-liquidated their position so the contract erased the position but failed to remove the associated order and collateral; (3) used the "ghost" collateral to borrow 6,260 ETH (~$12.9M). Economic invariants (collateral ↔ debt) were broken by the deposit-fail and liquidation path.  
  Key lessons:
  - All economic invariants (e.g., minimum collateralization, max LTV) must be **provable and enforced** on every state transition.
  - Introducing new spell/strategy types must go through **formal review** of how they interact with the existing system.  
  - [https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05](https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05)
  - [https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025](https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025)
  - [https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens](https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens)
  - [https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained](https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained)

- **Yearn Finance (November 2025, $9M loss)**  
  A design flaw in the yETH weighted stableswap pool's fixed-point iteration solver. By performing highly imbalanced add/remove liquidity operations, attackers forced the solver into a divergent regime, causing the product term (Π) to collapse to zero—converting the pool from a hybrid stableswap invariant to a constant-sum curve. This allowed minting ~2.35×10⁵⁶ yETH LP tokens without collateral and draining the pool. The vulnerability was **invariant collapse**, not reentrancy or low-level bugs.  
  Key lessons:
  - Reward and fee distribution paths must be **simulation-tested** across adversarial scenarios.
  - Yield strategies should have **clear, testable invariants** (e.g., no user can claim more rewards than their fair share over time).  
  - [https://defimon.xyz/blog/yearn-yeth-hack-november-2025](https://defimon.xyz/blog/yearn-yeth-hack-november-2025)
  - [https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/](https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/)

### Best Practices & Mitigations

- **Model protocol economics explicitly** (e.g., with adversarial simulations / agent-based models) rather than relying on intuition.
- Express **core invariants in code and tests**:
  - “Total value withdrawn cannot exceed total deposits + realized yield”
  - “Rewards distribution is proportional to time-weighted stake”
  - “Liquidations never result in protocol loss under honest oracle data”
- Use **formal verification and property-based fuzzing** for key accounting paths (vaults, strategies, reward distribution).
- **Version and gate new strategies / spells**:
  - Roll out behind caps.
  - Monitor metrics and on-chain invariants before raising limits.
- Ensure **governance and operations teams understand invariants**, not just auditors.

