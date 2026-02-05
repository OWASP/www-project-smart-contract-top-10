# SC02:2025 - 가격 오라클 조작

## 설명:
가격 오라클 조작은 외부 데이터 피드(오라클)를 사용하여 가격이나 기타 정보를 가져오는 스마트 컨트랙트의 심각한 취약점입니다. 탈중앙화 금융(DeFi)에서 오라클은 자산 가격과 같은 실제 세계 데이터를 스마트 컨트랙트에 제공하는 데 사용됩니다. 그러나 오라클이 제공하는 데이터가 조작되면 잘못된 컨트랙트 동작이 발생할 수 있습니다. 공격자는 오라클이 제공하는 데이터를 조작하여 무단 인출, 과도한 레버리지, 심지어 유동성 풀 고갈과 같은 파괴적인 결과를 초래할 수 있습니다. 이러한 유형의 공격을 방지하기 위해서는 적절한 보호 장치와 검증 메커니즘이 필수적입니다.

## 예시 (취약한 컨트랙트):

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

        // 취약점: 가격 조작에 대한 검증이나 보호가 없음
        uint256 collateralValue = uint256(price) * amount;

        // 조작된 가격을 기반으로 한 대출 로직
        // 공격자가 오라클을 조작하면 정당한 것보다 더 많이 대출받을 수 있음
    }

    function repay(uint256 amount) public {
        // 상환 로직
    }
}
```

### 영향:
- 공격자가 오라클을 조작하여 자산 가격을 부풀려 본래 받을 수 있는 것보다 더 많은 자금을 대출받을 수 있습니다.
- 조작된 가격이 담보의 잘못된 평가로 이어지는 경우, 정당한 사용자가 잘못된 가치 평가로 인해 청산에 직면할 수 있습니다.
- 오라클이 침해되면 공격자는 조작된 데이터를 악용하여 컨트랙트의 유동성 풀을 고갈시키거나 컨트랙트를 지불 불능 상태로 만들 수 있습니다.

### 해결 방안:
- 여러 개의 독립적인 오라클에서 데이터를 집계하여 단일 소스에 의한 조작 위험을 줄입니다.
- 오라클에서 받은 가격에 대해 최소 및 최대 임계값을 설정하여 급격한 가격 변동이 컨트랙트의 로직에 영향을 미치는 것을 방지합니다.
- 가격 업데이트 사이에 타임락을 도입하여 공격자가 악용할 수 있는 즉각적인 변경을 방지합니다.
- 신뢰할 수 있는 당사자의 서명을 요구하는 등 오라클에서 받은 데이터의 진위를 보장하기 위해 암호화 증명을 사용합니다.

### 예시 (수정된 버전):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // 최소 허용 가격 설정
    int public maxPrice = 2000; // 최대 허용 가격 설정

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // 유효한 가격을 기반으로 한 대출 로직
    }

    function repay(uint256 amount) public {
        // 상환 로직
    }
}
```

### 가격 오라클 조작 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Polter Finance 해킹 분석](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO 프로토콜](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) : 상세 [해킹 분석](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)
