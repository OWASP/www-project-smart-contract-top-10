## SC03:2026 - Price Oracle Manipulation

#### Description

Price oracle manipulation describes any situation where a smart contract relies on price or valuation data that can be **directly or indirectly influenced by an attacker**, causing the protocol to make decisions based on incorrect values. Oracles are trust boundaries: the contract implicitly trusts that the price it receives reflects real-world or on-chain market conditions. When that trust is violated—whether by manipulation, staleness, or misconfiguration—protocol behavior is distorted.

This affects all contract types that consume price data: DeFi lending and borrowing (collateral valuation, liquidation), AMMs and DEXes (spot and TWAP-based pricing), yield vaults (NAV calculations, share valuation), liquid staking and derivatives (ETH/stake price feeds), NFT and token valuations (floor price oracles), and cross-chain bridges (asset pricing for mint/burn ratios). On non-EVM chains (e.g., Move, Solana), similar patterns apply wherever external price sources feed into on-chain logic.

Few areas to focus on:

- **DEX-based oracles** (spot price, TWAP, geometric mean) and resistance to flash loans, JIT liquidity, or concentrated liquidity skew
- **Off-chain and hybrid feeds** (Chainlink, Pyth, custom relayers) and assumptions about freshness, deviation, and multi-source aggregation
- **Liquidity and market depth** of the underlying price source (thin pools vs. deep markets)
- **Cross-chain and L2 pricing** (finality delays, sequencer ordering, message relay assumptions)

Attackers exploit:

- **Spot price manipulation** via large trades, flash loans, or JIT liquidity in the same block
- **TWAP manipulation** over short windows or during low-liquidity periods
- **Stale or stuck data** when contracts do not enforce freshness or fallback behavior
- **Deviation and outlier handling** when aggregation logic fails to reject manipulated inputs

### Example (Vulnerable Oracle Usage)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IPriceFeed {
    function latestAnswer() external view returns (int256);
}

contract VulnerableOracleLending {
    IPriceFeed public priceFeed; // single-point oracle
    mapping(address => uint256) public collateralEth;
    mapping(address => uint256) public debtUsd;

    constructor(IPriceFeed _feed) {
        priceFeed = _feed;
    }

    function depositCollateral() external payable {
        collateralEth[msg.sender] += msg.value;
    }

    function borrow(uint256 amountUsd) external {
        int256 price = priceFeed.latestAnswer(); // no sanity checks, no delay
        require(price > 0, "bad price");

        uint256 collateralUsd = (collateralEth[msg.sender] * uint256(price)) / 1e8;
        // Allows borrowing up to 100% of collateral value – overly generous
        require(collateralUsd >= amountUsd, "insufficient collateral");

        debtUsd[msg.sender] += amountUsd;
        // transfer stablecoin (omitted)
    }
}
```

**Issues:**

- Single oracle source, no aggregation or sanity checks.
- No upper/lower bounds or deviation checks against past values.
- Economic parameters (100% LTV) make even minor manipulations profitable.

### Example (Hardened Oracle Integration)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IAggregatorV3 {
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}

contract RobustOracleLending {
    IAggregatorV3 public immutable priceFeed;
    mapping(address => uint256) public collateralEth;
    mapping(address => uint256) public debtUsd;

    uint256 public constant MAX_DELAY = 1 hours;
    uint256 public constant COLLATERAL_FACTOR_BPS = 7500; // 75%

    constructor(IAggregatorV3 _feed) {
        priceFeed = _feed;
    }

    function _getSafePrice() internal view returns (uint256) {
        (, int256 answer, , uint256 updatedAt, uint80 answeredInRound) =
            priceFeed.latestRoundData();
        require(answer > 0, "bad answer");
        require(updatedAt != 0 && block.timestamp - updatedAt <= MAX_DELAY, "stale price");
        require(answeredInRound != 0, "incomplete round");
        return uint256(answer);
    }

    function depositCollateral() external payable {
        require(msg.value > 0, "no collateral");
        collateralEth[msg.sender] += msg.value;
    }

    function borrow(uint256 amountUsd) external {
        uint256 price = _getSafePrice();
        uint256 collateralUsd = (collateralEth[msg.sender] * price) / 1e8;
        uint256 maxBorrow = (collateralUsd * COLLATERAL_FACTOR_BPS) / 10_000;
        require(debtUsd[msg.sender] + amountUsd <= maxBorrow, "exceeds limit");

        debtUsd[msg.sender] += amountUsd;
        // transfer stablecoin (omitted)
    }
}
```

**Security Improvements:**

- Uses a price feed interface with **round metadata** to reject stale or incomplete data.
- Applies conservative **collateral factors** and explicit borrowing limits.
- Encapsulates price fetch and validation in `_getSafePrice`, making reasoning and testing easier.

In 2025, pure oracle-only mega-exploits were less frequent, but oracle manipulation was often one component in **multi-vector attacks**.

### 2025 Case Studies

- **NGP Token (September 2025, ~$2M loss)**  
  The protocol's `getPrice()` function relied solely on DEX pair (Uniswap V2/PancakeSwap) reserve balances to calculate token price. Attackers used a flash loan to swap large amounts and manipulate reserves, skewing the oracle to show artificially low values, then bypassed purchase limits and cooldown protections to drain ~$2M. Oracle manipulation was the **root cause**.  
  - [https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96](https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96)
  - [https://hacken.io/insights/ngp-hack-explained/](https://hacken.io/insights/ngp-hack-explained/)
  - [https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/](https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/)

- **GMX (July 2025, $42M loss)**  
  While the primary root cause was reentrancy in `executeDecreaseOrder`, the attack flow involved **price feed manipulation** in conjunction: the attacker manipulated global average short price for Bitcoin downward (~57×), then used a flash loan to purchase GLP at artificially low prices and redeem at inflated prices. Oracle/pricing was a key **enabler** of the exploit.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### Best Practices & Mitigations

- **Aggregate multiple sources**:
  - Use median/mean of several DEXs / oracles.
  - Reject outliers and anomalous deviations.
- **Time-based defenses**:
  - Use TWAPs over sufficient windows to resist short-lived manipulations.
  - Reject prices older than a maximum staleness threshold.
- **Liquidity-aware design**:
  - Avoid basing core prices on **illiquid pools**.
  - Cap impact of a single pool/feed on global pricing.
- **Fail-safe behavior**:
  - On suspicious or unavailable data, **halt sensitive operations** (borrowing, liquidations).
  - Use circuit breakers and rate limiting on parameter changes.
- **Monitoring & alerting**:
  - Track price deviations between your oracle and reference markets.
  - Set automated alerts for out-of-band movements or stuck oracles.

