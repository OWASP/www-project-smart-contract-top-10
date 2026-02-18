## SC02:2026 - 비즈니스 로직 취약점

#### 설명

비즈니스 로직 취약점이란 개별 저수준 검사(예: 타입 안전성, 재진입 가드, 접근 제어)가 올바르더라도 스마트 컨트랙트의 *의도된* 경제적 또는 기능적 동작이 훼손될 수 있는 모든 상황을 말합니다. 이는 시스템의 규칙, 인센티브, 상태 전환, 불변량이 온체인에서 모델링되는 방식의 **설계 결함**입니다. 저수준 버그(오버플로, 재진입)와 달리, 비즈니스 로직 결함은 규칙 자체가 안전하지 않을 때 발생합니다—코드는 "명시된 대로 동작"하지만, 명시된 내용이 악용 가능한 결과를 허용합니다.

이는 모든 스마트 컨트랙트 도메인에 적용됩니다: DeFi(대출, AMM, 볼트, 수익 전략), NFT(발행 로직, 로열티, 마켓플레이스 메커니즘), DAO(투표, 위임, 제안 실행), 브리지(소각/발행 비대칭, 유동성 규칙), 게임(보상 분배, 공정성), 그리고 멀티홉 상태 전환이 창발적 취약점을 만드는 크로스체인/L2 시스템.

주요 점검 영역:

- 모듈 간 **불변량 위반** (예: 볼트 ↔ 전략 ↔ 게이지, 담보 ↔ 부채, 공급 ↔ 담보)
- **보상 및 수수료 로직** (이중 계산, 과소/과다 적립, 잘못된 수혜자)
- 우회 가능하거나 일관성 없이 적용되는 **자격 및 한도 검사** (차입 한도, 발행 한도, 청산 임계값)
- 불가능하거나 일관성 없는 상태에 도달하기 위한 조작적 행동 시퀀스를 허용하는 **경로 의존적 상태 머신**
- **크로스모듈 및 크로스체인 가정** (예: 브리지 유동성, L2 최종성, 메시지 순서)

공격자가 악용하는 방식:

- **일관성 없는 회계 간 차익거래** (볼트 vs. 전략, 내부 vs. 외부 잔액)
- **작업 순서 엣지 케이스** (불변량을 깨는 입금/출금/청구 시퀀스)
- **자격 우회** (예: 스테이킹 없이 보상 청구, 건전한 포지션 청산)
- 경제적으로 비합리적인 상태를 만드는 **파라미터 조작** (이자 곡선, 수수료, 담보 비율)

### 예시 (취약한 대출 로직)

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

    // 취약: 총액이 아닌 *새 금액*으로 차입 한도를 계산
    function borrow(uint256 amount) external {
        uint256 allowed = (amount * collateralFactorBps) / 10_000;
        require(allowed >= amount, "not enough collateral"); // 의미 없는 검사

        debt[msg.sender] += amount;
        // 풀에서 토큰 전송 (생략)
    }
}
```

**문제점:**

- `allowed`가 사용자의 담보 잔액이 아닌 *요청된 차입 금액*으로 계산됩니다.
- `allowed >= amount` 검사는 `collateralFactorBps >= 10_000`인 경우 항상 성립하며, 그렇지 않은 경우에도 이런 식으로 잘못 사용되면 단순한 동어반복이 되어 어떠한 경제적 불변량도 강제하지 못합니다.

### 예시 (수정: 불변량 기반 차입 로직)

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
        uint256 price = oracle.getCollateralPrice(); // 예: USD 기준 ETH 가격 1e18
        uint256 collateralUsd = (collateral[user] * price) / 1e18;
        return (collateralUsd * collateralFactorBps) / 10_000;
    }

    function borrow(uint256 amountUsd) external {
        uint256 maxBorrowUsd = _maxBorrow(msg.sender);
        require(debt[msg.sender] + amountUsd <= maxBorrowUsd, "exceeds limit");

        debt[msg.sender] += amountUsd;
        // 풀에서 스테이블코인 전송 (생략)
    }
}
```

**보안 개선 사항:**

- 차입 한도가 요청 금액이 아닌 **총 담보**에서 도출됩니다.
- 경제적 불변량: `debt[user] <= maxBorrow(user)`가 모든 차입 시 강제됩니다.
- 가격 오라클이 명시적으로 통합됩니다 (SC03에 따라 강화 가능).

### 2025년 사례 연구

- **Abracadabra (2025년 3월, $1,290만 손실)**  
  GMX V2 CauldronV4 컨트랙트의 결함 있는 담보 회계. 공격자는 3단계 방법을 사용했습니다: (1) GMX에 실패하도록 설계된 입금을 수행하여 OrderAgent에 토큰을 묶어 두었습니다; (2) 자신의 포지션을 자기 청산하여 컨트랙트가 포지션을 삭제했지만 관련 주문과 담보를 제거하지 못하게 했습니다; (3) "유령" 담보를 사용하여 6,260 ETH(약 $1,290만)를 차입했습니다. 경제적 불변량(담보 ↔ 부채)이 입금 실패 및 청산 경로에 의해 깨졌습니다.  
  핵심 교훈:
  - 모든 경제적 불변량(예: 최소 담보화, 최대 LTV)은 모든 상태 전환에서 **증명 가능하고 강제**되어야 합니다.
  - 새로운 스펠/전략 유형 도입은 기존 시스템과의 상호작용에 대한 **공식 검토**를 거쳐야 합니다.  
  - [https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05](https://blog.solidityscan.com/abracadabra-hack-analysis-f2efcdee9c05)
  - [https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025](https://www.halborn.com/blog/post/explained-the-abracadabra-money-hack-march-2025)
  - [https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens](https://www.coindesk.com/business/2025/03/25/abracadabra-drained-of-usd13m-in-exploit-targeting-cauldrons-tied-to-gmx-liquidity-tokens)
  - [https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained](https://threesigma.xyz/blog/exploit/abracadabra-gmx-defi-exploit-explained)

- **Yearn Finance (2025년 11월, $900만 손실)**  
  yETH 가중 스테이블스왑 풀의 고정소수점 반복 솔버의 설계 결함. 공격자는 극도로 불균형한 유동성 추가/제거 작업을 수행하여 솔버를 발산 상태로 몰아넣어 곱 항(Π)이 0으로 붕괴되게 했습니다. 이로 인해 풀이 하이브리드 스테이블스왑 불변량에서 상수합 곡선으로 전환되었습니다. 이를 통해 담보 없이 약 2.35×10⁵⁶ yETH LP 토큰을 발행하고 풀을 소진할 수 있었습니다. 취약점은 재진입이나 저수준 버그가 아닌 **불변량 붕괴**였습니다.  
  핵심 교훈:
  - 보상 및 수수료 분배 경로는 적대적 시나리오에서 **시뮬레이션 테스트**되어야 합니다.
  - 수익 전략은 **명확하고 테스트 가능한 불변량**을 가져야 합니다 (예: 어떤 사용자도 시간 가중 지분보다 더 많은 보상을 청구할 수 없음).  
  - [https://defimon.xyz/blog/yearn-yeth-hack-november-2025](https://defimon.xyz/blog/yearn-yeth-hack-november-2025)
  - [https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/](https://www.cryptopolitan.com/yearn-finance-begins-clawback-after-9m-hack/)

### 모범 사례 및 완화 방법

- 직관에 의존하지 않고 **프로토콜 경제를 명시적으로 모델링**합니다 (예: 적대적 시뮬레이션 / 에이전트 기반 모델 사용).
- **핵심 불변량을 코드와 테스트로 표현**합니다:
  - "총 출금 금액은 총 입금액 + 실현 수익을 초과할 수 없다"
  - "보상 분배는 시간 가중 지분에 비례한다"
  - "청산은 정직한 오라클 데이터 하에서 프로토콜 손실을 초래하지 않는다"
- 핵심 회계 경로(볼트, 전략, 보상 분배)에 **공식 검증 및 속성 기반 퍼징**을 사용합니다.
- **새 전략 / 스펠을 버전 관리하고 게이팅**합니다:
  - 한도 뒤에서 출시합니다.
  - 한도를 높이기 전에 지표와 온체인 불변량을 모니터링합니다.
- **거버넌스 및 운영 팀이 불변량을 이해**하도록 합니다—감사자만이 아닌.

