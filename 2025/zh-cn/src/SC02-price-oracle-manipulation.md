# SC02:2025 - 价格预言机操纵

## 漏洞描述:
预言机价格操纵是智能合约中的一个关键漏洞，这些合约依赖外部数据源（预言机）来获取价格或其他信息。在去中心化金融（DeFi）中，预言机用于向智能合约提供真实世界的数据，如资产价格。然而，如果预言机提供的数据被操控，可能导致合约执行错误。攻击者可以通过操纵预言机提供的数据来发起攻击，进而导致严重后果，例如未授权提款、过度杠杆或甚至耗尽流动性池。因此，合理的保障措施和验证机制对于防止此类攻击至关重要。

## 举例 (包含漏洞的合约):

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

        // 漏洞: 未对价格数据进行验证或防范操纵
        uint256 collateralValue = uint256(price) * amount;

        // 基于被操纵价格的借贷逻辑
        // 如果攻击者操纵了预言机，他们可能会借出超过应有额度的资产
    }

    function repay(uint256 amount) public {
        // 还款逻辑
    }
}
```

### 漏洞影响:
- 攻击者可能操纵预言机以抬高资产价格，从而能够借入比原本有权获得的更多资金。
- 当被操纵的价格导致对抵押品的错误评估时，合法用户可能会因不准确的估值而面临清算。
- 如果预言机被攻破，攻击者可以利用被操控的数据来耗尽合约的流动性池，甚至导致合约资不抵债。

### 修复方法:
- 从多个独立的预言机聚合数据，以降低被任何单一来源操控的风险。
- 为从预言机接收的价格设置最低和最高阈值，防止剧烈的价格波动影响合约逻辑。
- 在价格更新之间引入时间锁，防止攻击者利用即时变化进行操控。
- 使用加密证明确保从预言机接收的数据的真实性，例如要求可信方提供签名验证。

### 举例 (已修复版本):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // 设定最低可接受价格  
    int public maxPrice = 2000; // 设定最高可接受价格

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // 基于有效价格的借贷逻辑
    }

    function repay(uint256 amount) public {
        // 还款逻辑
    }
}
```

### 遭受价格预言机操纵攻击的案例 :
1. [Polter Finance Hack Analysis](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO Protocol](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)
