# SC02:2025 - Price Oracle Manipulation

## Description:
Price Oracle Manipulation is a critical vulnerability in smart contracts that rely on external data feeds (oracles) to fetch prices or other information. In decentralized finance (DeFi), oracles are used to provide real-world data, such as asset prices, to smart contracts. However, if the data provided by the oracle is manipulated, it can result in incorrect contract behavior. Attackers can exploit oracles by manipulating the data they supply, leading to devastating consequences such as unauthorized withdrawals, excessive leverage, or even draining liquidity pools. Proper safeguards and validation mechanisms are essential to prevent this type of attack.
Example (Vulnerable contract):

## Example (Vulnerable Contract):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0, "Price must be positive");

        // Vulnerability: No validation or protection against price manipulation
        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on manipulated price
        // If an attacker manipulates the oracle, they could borrow more than they should
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```

### Impact:
- Attackers could manipulate the oracle to inflate the price of an asset, allowing them to borrow more funds than they would otherwise be entitled to.
- In cases where the manipulated price leads to a false assessment of collateral, legitimate users may face liquidation due to incorrect valuations.
- If an oracle is compromised, attackers can exploit the manipulated data to drain the contract’s liquidity pools or even cause a contract to become insolvent.

### Remediation:
- Aggregate data from multiple, independent oracles to reduce the risk of manipulation by any single source.
- Set minimum and maximum thresholds for the prices received from the oracle to prevent drastic price swings from affecting the contract’s logic.
- Introduce a time lock between price updates to prevent instant changes that could be exploited by attackers.
- Use cryptographic proofs to ensure the authenticity of data received from oracles, such as requiring signatures from trusted parties.

### Example (Fixed version):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // Set minimum acceptable price
    int public maxPrice = 2000; // Set maximum acceptable price

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on valid price
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```

### Examples of Smart Contracts that fell victim to Price Oracle Manipulation Attacks :
1. [Polter Finance Hack Analysis](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO Protocol](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)
