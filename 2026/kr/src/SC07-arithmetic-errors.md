## SC07:2026 - 산술 오류 (반올림 및 정밀도)

#### 설명

산술 오류(반올림 및 정밀도 손실)란 스마트 컨트랙트가 절사, 스케일링, 또는 단위 변환으로 인해 잘못되거나 악용 가능한 결과를 생성하는 정수 기반 계산을 수행하는 모든 상황을 말합니다. 스마트 컨트랙트는 정수 산술로 제한됩니다; 나눗셈, 고정소수점 스케일링, 또는 단위 간 변환은 정밀도를 잃거나, 비대칭 반올림을 도입하거나—unchecked 블록 또는 비EVM 의미론과 결합될 때—오버플로/언더플로를 유발할 수 있습니다 (SC09 참조).

이는 수치 값을 계산하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi(지분 발행/소각, LP 토큰, 이자 적립, 스왑 출력, AMM 불변량 업데이트), 수익 볼트 및 ERC-4626(자산/지분 변환), 리베이싱 토큰, 보상 분배, NFT/토큰 경제학. 비EVM 체인(예: Move, Rust 기반)에서는 정수 의미론과 가용 정밀도가 다릅니다; 산술이 경제적 결과를 이끄는 곳이라면 유사한 위험이 적용됩니다.

주요 점검 영역:

- **지분 및 LP 토큰 계산** (입금/출금 공식, 반올림 방향)
- **이자 및 보상 적립** (복리, 시간 가중 평균)
- **스왑 및 AMM 수학** (상수 곱, 집중 유동성, 출력 계산)
- **고정소수점 및 스케일링** (1e18, 1e8 관례, 크로스 토큰 변환)
- **리베이싱 및 비례 분배** (사용자별 vs. 글로벌 회계)

공격자가 악용하는 방식:

- **반올림 편향** (예: 적대적 시퀀스 하에서 입금자 또는 프로토콜에 유리한 반올림)
- 플래시 론(SC04) 또는 고빈도 상호작용을 통한 **반복적인 소규모 이득**
- 공식이 무너지는 **엣지 케이스** (총 공급 영, 첫 입금자, 극단적 비율)
- 작업 전반에 걸쳐 누적되는 다단계 계산의 **정밀도 손실**

플래시 론(SC04) 또는 비즈니스 로직 결함(SC02)과 결합되면 산술 오류는 프로토콜 소진 익스플로잇으로 증폭될 수 있습니다.

### 예시 (취약한 지분 계산)

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
            // 내림 반올림으로 특정 엣지 상태에서 입금자에게 유리할 수 있음
            shares = (assets * totalShares) / totalAssets;
        }

        totalAssets += assets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**문제점:**

- 반올림이 항상 절사됩니다; 적대적인 입금/출금 시퀀스 하에서 이를 게임화할 수 있습니다.
- 엣지 케이스 전반에 걸쳐 총 지분 가치가 일관성을 유지하는지 확인하는 불변량 테스트가 없습니다.

### 예시 (더 견고한 산술 및 불변량 인식 설계)

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
            // 초기 조건을 명시적으로 정의
            shares = assets;
        } else {
            // 의도적으로 반올림 전략 선택 (올림 또는 내림) 및 테스트
            shares = (assets * totalShares + totalAssets - 1) / totalAssets; // 올림
        }

        uint256 newTotalAssets = totalAssets + assets;
        if (newTotalAssets < totalAssets) revert InvalidState(); // 오버플로 가드

        totalAssets = newTotalAssets;
        totalShares += shares;
        balanceOf[msg.sender] += shares;
    }
}
```

**보안 개선 사항:**

- 초기 상태를 명시적으로 처리하고 영으로 나누기를 방지합니다.
- 명확히 문서화된 반올림 전략 사용 (이 예시에서는 `올림`).
- 명확성을 위해 오버플로 검사와 커스텀 에러 추가.

> 실제 프로토콜은 이러한 로직을 공식적 추론 및 불변량과 함께 사용해야 합니다 (예: 공식 검증 또는 속성 기반 테스트를 통해).

### 2025년 사례 연구

- **zkLend (2025년 2월, $950만 손실)**  
  `mint()` 함수의 반올림 오류—정수 나눗셈 내림 반올림—로 인해 기록된 값과 실제 값 사이에 불일치가 발생했습니다. 1.5 토큰을 소각해야 할 출금이 1.0으로 내림되어 공격자가 반복적인 입금/출금을 통해 lending_accumulator를 인위적으로 부풀리고 약 $950만을 소진할 수 있었습니다.  
  - [https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71](https://blog.solidityscan.com/zklend-hack-analysis-e494cb794f71)
  - [https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025](https://www.halborn.com/blog/post/explained-the-zklend-hack-february-2025)

- **Bunni (2025년 9월, $840만 손실)**  
  출금 함수의 반올림 로직의 정밀도 버그. 개발자들은 유휴 잔액을 내림 반올림하는 것이 "안전"하다고 가정했지만, 반복 작업이 악용 가능한 허점을 만들었습니다—공격자는 유동성의 84.4%만 소각하면서 USDC 잔액을 85.7% 감소시켜 불균형한 가치 추출을 가능하게 했습니다.  
  - [https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025](https://www.halborn.com/blog/post/explained-the-bunni-hack-september-2025)
  - [https://blog.bunni.xyz/posts/exploit-post-mortem/](https://blog.bunni.xyz/posts/exploit-post-mortem/)
  - [https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit](https://cryptotimes.io/2025/09/05/bunni-reveals-code-flaw-behind-8-4-million-exploit)

### 모범 사례 및 완화 방법

- **안전한 수학 패턴 사용** (Solidity 0.8+는 내장 검사를 제공하지만 그 주변의 로직도 중요합니다).
- **반올림 전략**을 명확히 문서화하고 테스트합니다:
  - 반올림이 프로토콜 또는 사용자에게 유리해야 하는지 결정합니다.
  - 반복 상호작용이 "무료 가치"를 만들 수 없음을 증명합니다.
- 복잡한 작업에 **잘 검토된 수학 라이브러리**에 의존합니다:
  - 고정소수점 수학 (예: 1e18 스케일링)
  - 고정밀 지수화 또는 로그
- **불변량 검사** 통합:
  - 예: 작업 후 `totalAssets` vs. 사용자 잔액의 합, 또는 지분/가치 일관성.
- 소규모/대규모 값과 반복 작업 주변의 엣지 케이스를 발견하기 위해 **퍼즈 테스팅 및 차분 테스팅** 사용.

