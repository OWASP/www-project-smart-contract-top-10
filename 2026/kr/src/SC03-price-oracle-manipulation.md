## SC03:2026 - 가격 오라클 조작

#### 설명

가격 오라클 조작이란 스마트 컨트랙트가 **공격자가 직접 또는 간접적으로 영향을 미칠 수 있는** 가격 또는 가치 평가 데이터에 의존하여, 프로토콜이 잘못된 값을 기반으로 결정을 내리게 되는 모든 상황을 말합니다. 오라클은 신뢰 경계입니다: 컨트랙트는 수신하는 가격이 실제 세계 또는 온체인 시장 상황을 반영한다고 암묵적으로 신뢰합니다. 조작, 오래된 데이터, 또는 잘못된 구성으로 인해 이 신뢰가 침해되면 프로토콜 동작이 왜곡됩니다.

이는 가격 데이터를 소비하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi 대출 및 차입(담보 가치 평가, 청산), AMM 및 DEX(현물 및 TWAP 기반 가격 책정), 수익 볼트(NAV 계산, 지분 가치 평가), 유동 스테이킹 및 파생상품(ETH/스테이크 가격 피드), NFT 및 토큰 가치 평가(바닥가 오라클), 크로스체인 브리지(발행/소각 비율을 위한 자산 가격 책정). 비EVM 체인(예: Move, Solana)에서도 외부 가격 소스가 온체인 로직에 공급되는 곳이라면 유사한 패턴이 적용됩니다.

주요 점검 영역:

- **DEX 기반 오라클** (현물 가격, TWAP, 기하 평균) 및 플래시 론, JIT 유동성, 집중 유동성 왜곡에 대한 저항성
- **오프체인 및 하이브리드 피드** (Chainlink, Pyth, 커스텀 릴레이어) 및 신선도, 편차, 다중 소스 집계에 대한 가정
- 기반 가격 소스의 **유동성 및 시장 깊이** (얕은 풀 vs. 깊은 시장)
- **크로스체인 및 L2 가격 책정** (최종성 지연, 시퀀서 순서, 메시지 릴레이 가정)

공격자가 악용하는 방식:

- 동일 블록 내 대규모 거래, 플래시 론, 또는 JIT 유동성을 통한 **현물 가격 조작**
- 짧은 윈도우 또는 낮은 유동성 기간 동안의 **TWAP 조작**
- 컨트랙트가 신선도 또는 폴백 동작을 강제하지 않을 때의 **오래되거나 고착된 데이터**
- 집계 로직이 조작된 입력을 거부하지 못할 때의 **편차 및 이상값 처리**

### 예시 (취약한 오라클 사용)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IPriceFeed {
    function latestAnswer() external view returns (int256);
}

contract VulnerableOracleLending {
    IPriceFeed public priceFeed; // 단일 포인트 오라클
    mapping(address => uint256) public collateralEth;
    mapping(address => uint256) public debtUsd;

    constructor(IPriceFeed _feed) {
        priceFeed = _feed;
    }

    function depositCollateral() external payable {
        collateralEth[msg.sender] += msg.value;
    }

    function borrow(uint256 amountUsd) external {
        int256 price = priceFeed.latestAnswer(); // 건전성 검사 없음, 지연 없음
        require(price > 0, "bad price");

        uint256 collateralUsd = (collateralEth[msg.sender] * uint256(price)) / 1e8;
        // 담보 가치의 100%까지 차입 허용 – 지나치게 관대함
        require(collateralUsd >= amountUsd, "insufficient collateral");

        debtUsd[msg.sender] += amountUsd;
        // 스테이블코인 전송 (생략)
    }
}
```

**문제점:**

- 단일 오라클 소스, 집계 또는 건전성 검사 없음.
- 과거 값 대비 상한/하한 또는 편차 검사 없음.
- 경제적 파라미터(100% LTV)로 인해 사소한 조작도 수익성이 있습니다.

### 예시 (강화된 오라클 통합)

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
        // 스테이블코인 전송 (생략)
    }
}
```

**보안 개선 사항:**

- 오래되거나 불완전한 데이터를 거부하기 위해 **라운드 메타데이터**가 포함된 가격 피드 인터페이스를 사용합니다.
- 보수적인 **담보 비율**과 명시적인 차입 한도를 적용합니다.
- 가격 가져오기 및 검증을 `_getSafePrice`에 캡슐화하여 추론과 테스트를 용이하게 합니다.

2025년에는 순수 오라클 단독 대규모 익스플로잇은 덜 빈번했지만, 오라클 조작은 종종 **다중 벡터 공격**의 한 구성 요소였습니다.

### 2025년 사례 연구

- **NGP Token (2025년 9월, 약 $200만 손실)**  
  프로토콜의 `getPrice()` 함수가 토큰 가격 계산을 위해 DEX 페어(Uniswap V2/PancakeSwap) 준비금 잔액에만 의존했습니다. 공격자는 플래시 론을 사용하여 대량 스왑으로 준비금을 조작하고 오라클이 인위적으로 낮은 값을 표시하게 한 다음, 구매 한도와 쿨다운 보호를 우회하여 약 $200만을 소진했습니다. 오라클 조작이 **근본 원인**이었습니다.  
  - [https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96](https://blog.solidityscan.com/ngp-token-hack-analysis-414b6ca16d96)
  - [https://hacken.io/insights/ngp-hack-explained/](https://hacken.io/insights/ngp-hack-explained/)
  - [https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/](https://coincentral.com/flash-loan-exploit-drains-2-million-from-ngp-token-on-bnb-chain/)

- **GMX (2025년 7월, $4,200만 손실)**  
  주요 근본 원인은 `executeDecreaseOrder`의 재진입이었지만, 공격 흐름에는 **가격 피드 조작**도 포함되었습니다: 공격자는 비트코인의 글로벌 평균 숏 가격을 약 57배 하락시킨 다음, 플래시 론을 사용하여 인위적으로 낮은 가격에 GLP를 구매하고 부풀려진 가격에 환매했습니다. 오라클/가격 책정이 익스플로잇의 핵심 **조력자**였습니다.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### 모범 사례 및 완화 방법

- **여러 소스 집계**:
  - 여러 DEX / 오라클의 중앙값/평균 사용.
  - 이상값 및 비정상적인 편차 거부.
- **시간 기반 방어**:
  - 단기 조작에 저항하기 위해 충분한 윈도우의 TWAP 사용.
  - 최대 허용 경과 시간을 초과한 가격 거부.
- **유동성 인식 설계**:
  - **비유동성 풀**을 기반으로 핵심 가격 책정 회피.
  - 단일 풀/피드가 글로벌 가격 책정에 미치는 영향 제한.
- **안전 장치 동작**:
  - 의심스럽거나 사용 불가능한 데이터 시 **민감한 작업 중단** (차입, 청산).
  - 파라미터 변경에 서킷 브레이커 및 속도 제한 사용.
- **모니터링 및 알림**:
  - 오라클과 참조 시장 간의 가격 편차 추적.
  - 범위 외 움직임 또는 고착된 오라클에 대한 자동 알림 설정.

