## SC04:2026 - 플래시 론 기반 공격

#### 설명

플래시 론 기반 공격이란 공격자가 **무담보, 동일 트랜잭션 내 차입**(플래시 론)을 사용하여 기반 취약점을 수익성 있는 프로토콜 소진 공격으로 증폭시키는 익스플로잇을 말합니다. 플래시 론 자체는 취약하지 않습니다—이는 합법적인 DeFi 기본 요소입니다—하지만 공격자에게 단일 트랜잭션 내에서 자신의 자금을 위험에 노출시키지 않고도 임의로 큰 일시적 자본을 부여합니다. *위험에 처한 자본* 또는 *역사적 포지션 크기*에 기반한 신뢰 가정을 하는 모든 프로토콜은 공격자가 자신의 자금 없이 일시적으로 막대한 잔액을 보유할 수 있을 때 노출됩니다.

이는 경제적 제약이나 비례적 노출을 가정하는 모든 컨트랙트 유형에 영향을 미칩니다: 대출 프로토콜(거버넌스 투표 가중치, 청산 임계값, 담보 비율), AMM(지분 발행, 차익거래, 오라클 왜곡), 수익 볼트(입금/출금/지분 회계), 거버넌스(투표 매수, 플래시 론 거버넌스 공격), NFT 및 토큰 가치 평가(바닥가, 담보 가치 평가), 그리고 한 컨트랙트의 상태가 다른 컨트랙트의 유동성에 의해 영향받는 크로스 프로토콜 구성 가능성. 비EVM 체인에서도 유사한 개념이 존재합니다 (예: 단일 블록 또는 배치 내 일시적 대규모 포지션).

주요 점검 영역:

- **거버넌스 및 투표** (플래시 론 투표 매수, 스냅샷 조작)
- **오라클 및 가격 책정** (차입 유동성으로 DEX/TWAP 피드 조작)
- **지분 및 회계 로직** (반올림, 제한된 입력을 가정하는 비례 계산)
- **청산 및 담보 검사** (포지션 크기 의존 임계값)
- **구성 가능성** (단일 트랜잭션에서 호출자 잔액 또는 풀 상태를 신뢰하는 프로토콜)

공격자는 다음과 같이 배치된 트랜잭션을 구성합니다:

1. 플래시 론(Aave, dYdX, Uniswap V3 플래시, 또는 동등한 것)을 통해 대규모 자본 차입.
2. 차입 자금을 사용하여 프로토콜 상태, 가격, 또는 회계 조작.
3. 이익 추출 (유동성 소진, 미담보 대출 취득, 거버넌스 왜곡).
4. 동일 트랜잭션에서 플래시 론 상환, 이익 보유.

비즈니스 로직(SC02), 오라클(SC03), 산술(SC07), 또는 접근 제어(SC01)의 기반 약점이 존재할 때, 플래시 론은 **힘 증폭기** 역할을 하여 작은 버그를 치명적인 익스플로잇으로 전환합니다.

### 예시 (반올림 버그가 있는 플래시 론의 취약한 사용)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IFlashLender {
    function flashLoan(
        address receiver,
        uint256 amount,
        bytes calldata data
    ) external;
}

contract VulnerablePool {
    uint256 public totalShares;
    uint256 public totalAssets;

    mapping(address => uint256) public sharesOf;

    // 취약: 발신자에게 유리한 절사가 있는 단순한 지분 발행
    function deposit(uint256 assets) external {
        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            shares = (assets * totalShares) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        sharesOf[msg.sender] += shares;
    }

    // 플래시 론 기반 입금/출금 루프에 대한 안전장치 없음
}
```

**문제점:**

- 반올림은 항상 프로토콜에 유리하게 절사되지만, 반복적인 플래시 론 기반 사이클에서 잘못 조정된 공식이나 잘못 계산된 상태는 공격자에게 순이익으로 전환될 수 있습니다.
- **최대 슬리피지, 한도, 또는 빈도**에 대한 고려가 없어 반복적인 대량 작업이 가능합니다.

### 예시 (완화된 설계 고려사항)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SaferPool {
    uint256 public totalShares;
    uint256 public totalAssets;
    uint256 public lastUpdateBlock;

    mapping(address => uint256) public sharesOf;

    error TooFrequentInteraction();

    modifier rateLimited() {
        // 간단한 예시: 고영향 작업을 블록당 한 번으로 제한
        if (lastUpdateBlock == block.number) revert TooFrequentInteraction();
        _;
        lastUpdateBlock = block.number;
    }

    function deposit(uint256 assets) external rateLimited {
        require(assets > 0, "zero assets");

        uint256 shares;
        if (totalShares == 0) {
            shares = assets;
        } else {
            // 프로토콜에 유리하게 반올림하고 공식적으로 분석된 반올림 사용
            shares = (assets * totalShares + totalAssets - 1) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        sharesOf[msg.sender] += shares;
    }
}
```

**보안 개선 사항:**

- 빠른 플래시 론 루프에 대한 취약성을 줄이기 위해 **기본 속도 제한** 도입.
- 더 보수적이고 명시적으로 문서화된 반올림 전략 사용.
- 지분/회계 공식과 불변량의 공식 분석 권장.

> 참고: 실제 프로토콜은 블록당 한도에만 의존하지 않고 더 강력한 심층 방어 메커니즘을 사용해야 합니다.

### 2025년 사례 연구

- **Bunni (2025년 9월, $840만 손실)**  
  출금 함수의 반올림 오류가 플래시 론으로 증폭되었습니다. 공격자는 300만 USDT를 플래시 차입하여 풀의 현물 가격을 극단으로 밀어붙이고(USDC 활성 잔액을 28 wei로), 반올림을 악용하는 44개의 연쇄된 소규모 출금을 실행했습니다—USDC 잔액을 85.7% 감소시키면서 유동성의 84.4%만 소각했습니다. 플래시 론이 필요한 자본 규모를 가능하게 했습니다.  
  - [https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025](https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025)
  - [https://blog.bunni.xyz/posts/exploit-post-mortem/](https://blog.bunni.xyz/posts/exploit-post-mortem/)
  - [https://cryptorank.io/news/feed/27dc8-bunni-hit-by-8-4m-flash-loan-exploit-rounding-error-blamed](https://cryptorank.io/news/feed/27dc8-bunni-hit-by-8-4m-flash-loan-exploit-rounding-error-blamed)

- **zkLend (2025년 2월, $950만 손실)**  
  `mint()` 함수의 반올림 오류(정수 나눗셈 내림 반올림)로 인해 공격자가 반복적인 입금/출금을 통해 lending_accumulator를 부풀릴 수 있었습니다. 플래시 론이 포지션을 확대하여 반복당 작은 정밀도 이득을 약 $950만 소진으로 전환했습니다. 플래시 론이 **힘 증폭기**였습니다.  
  - [https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71](https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71)
  - [https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025](https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025)
  - [https://zircon.tech/blog/the-9-5m-zklend-hack-another-defi-security-wake-up-call/](https://zircon.tech/blog/the-9-5m-zklend-hack-another-defi-security-wake-up-call/)

### 모범 사례 및 완화 방법

- **플래시 론이 존재한다고 가정**: 임의로 크고 일시적인 자본이 공격자에게 가용하다는 가정 하에 경제적 및 회계 로직을 설계합니다.
- **민감한 작업에 속도 제한 적용**:
  - 리베이싱, 리밸런싱, 또는 고영향 상태 전환에 대한 블록당 또는 에포크당 한도.
  - 행동의 규모/빈도에 따라 증가하는 동적 수수료.
- **상호작용당 노출 제한**:
  - 최대 슬리피지, 최대 포지션 크기, 차입 한도 설정.
  - 단일 트랜잭션에서 변경 가능한 상태의 양 제한.
- **플래시 론 시나리오 시뮬레이션**:
  - QA 및 감사에 플래시 론 스타일 테스트 포함.
  - 퍼징을 사용하여 수익성 있는 멀티콜 시퀀스 발견.
- **강력한 오라클 및 로직과 결합**:
  - 플래시 론은 보통 *증폭기*입니다; 기반 문제(SC02, SC03, SC07)는 근본에서 수정되어야 합니다.

