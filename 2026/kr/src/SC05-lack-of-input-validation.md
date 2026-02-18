## SC05:2026 - 입력값 검증 부재

#### 설명

입력값 검증 부재란 스마트 컨트랙트가 외부 데이터—함수 파라미터, 콜데이터, 크로스체인 메시지, 또는 서명된 페이로드—를 데이터가 올바른 형식인지, 예상 범위 내에 있는지, 의도된 작업에 대해 권한이 있는지를 **엄격하게 강제하지 않고** 처리하는 모든 상황을 말합니다. 입력이 무해하다고 가정하는 컨트랙트는 시스템을 안전하지 않은 상태로 밀어넣거나, 회계를 손상시키거나, 의도된 검사를 우회하는 잘못된 형식 또는 적대적 데이터에 노출됩니다.

이는 모든 컨트랙트 유형에 적용됩니다: DeFi(수수료 bps, 슬리피지, 금액, 주소), NFT(토큰 ID, 메타데이터, 로열티 구성), DAO(제안 페이로드, 투표 파라미터), 브리지(메시지 페이로드, 목적지 체인), 그리고 임의의 콜데이터 또는 릴레이된 호출을 수락하는 일반 구성 가능한 컨트랙트. 비EVM 체인에서도 동일한 원칙이 적용됩니다: 사용자, 다른 컨트랙트, 또는 크로스체인 채널로부터의 신뢰할 수 없는 입력은 사용 전에 검증되어야 합니다.

주요 점검 영역:

- **숫자 파라미터** (금액, 수수료, 이율, 슬리피지, 담보 비율) 및 안전한 범위
- **주소** (제로 주소, 컨트랙트 vs. EOA 가정, 위임 또는 프록시 주소)
- **오프체인 및 서명된 데이터** (서명, 만료, 논스 재사용)
- **크로스체인 및 브리지 페이로드** (메시지 형식, 체인 ID, 발신자 검증)
- **관리자 및 거버넌스 입력** (구성 값, 업그레이드 파라미터)—종종 신뢰할 수 있는 것으로 처리되지만 잘못 구성되거나 악용될 수 있음

공격자가 악용하는 방식:

- 불변량을 깨는 **범위 외 값** (예: 수수료 > 100%, 0 금액, 최대 uint)
- 허용 목록을 우회하거나 예상치 못한 동작을 유발하는 **잘못된 형식의 주소 또는 페이로드**
- 논스/만료/체인 ID가 검증되지 않을 때의 **재사용 및 순서 공격**
- 컨트랙트가 호출자 형식을 가정하거나 릴레이된 데이터를 신뢰할 때의 **구성 가능성 엣지 케이스**

2025년에는 입력값 검증 문제가 종종 *기여 요인*으로 나타났습니다. 예를 들어, 유동성 또는 이자 계산을 제어하는 파라미터에 안전한 범위를 강제하지 않는 경우.

### 예시 (취약한 파라미터 처리)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableConfig {
    uint256 public feeBps;      // 기준점 0–10_000
    uint256 public maxDeposit;  // 입금 상한

    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        // 누락: 접근 제어 및 범위 검사
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }
}
```

**문제점:**

- 접근 제어 없음: 누구나 `setConfig`를 호출할 수 있습니다.
- `_feeBps` 또는 `_maxDeposit`에 대한 검증 없음:
  - `feeBps`가 100%를 초과하여 수수료 로직을 깨뜨릴 수 있습니다.
  - `maxDeposit`이 안전하지 않거나 영 값으로 설정되어 프로토콜을 방해할 수 있습니다.

### 예시 (강력한 검증이 적용된 수정 버전)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract SafeConfig {
    uint256 public feeBps;      // 0–1_000 (최대 10% 수수료)
    uint256 public maxDeposit;  // 입금 상한

    address public immutable owner;

    error NotOwner();
    error InvalidFee();
    error InvalidMaxDeposit();

    constructor(uint256 initialFeeBps, uint256 initialMaxDeposit) {
        owner = msg.sender;
        _setConfig(initialFeeBps, initialMaxDeposit);
    }

    function _setConfig(uint256 _feeBps, uint256 _maxDeposit) internal {
        if (_feeBps > 1_000) revert InvalidFee();
        if (_maxDeposit == 0) revert InvalidMaxDeposit();
        feeBps = _feeBps;
        maxDeposit = _maxDeposit;
    }

    function setConfig(uint256 _feeBps, uint256 _maxDeposit) external {
        if (msg.sender != owner) revert NotOwner();
        _setConfig(_feeBps, _maxDeposit);
    }
}
```

**보안 개선 사항:**

- 수수료가 문서화된 안전한 범위 내에 있는지 검증합니다.
- `maxDeposit`이 0이 아닌 값이도록 요구하여 잘못된 구성을 방지합니다.
- 구성 변경을 컨트랙트 소유자로 제한합니다 (더 고급 RBAC는 SC01 참조).

### 2025년 사례 연구

- **Cetus (2025년 5월, $2억 2,300만 손실)**  
  주요 근본 원인은 `checked_shlw`의 결함 있는 오버플로 검사였습니다 (SC09 참조). 그러나 **불충분한 입력값 검증**이 기여 요인이었습니다—프로토콜이 범위 검사 없이 극단적인 유동성 파라미터(예: ~2^113)를 허용했습니다. 결함 있는 산술과 결합되어 이러한 검증되지 않은 입력이 풀 소진으로 이어지는 위험한 엣지 케이스를 만들었습니다.  
  - [https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
  - [https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)
  - [https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025)

- **Ionic Money (2025년 2월, 약 $690만 손실)**  
  공격자는 **소셜 엔지니어링**을 사용하여 프로토콜이 위조 LBTC 토큰을 등록하도록 설득했습니다. 일단 등록되면 프로토콜은 토큰 진위를 온체인에서 검증하지 않고 담보로 수락했습니다 (예: 등록된 담보 컨트랙트가 합법적인지 검증). 공격자는 250개의 가짜 LBTC를 발행하고 이를 사용하여 약 $860만을 차입했습니다. *참고: 근본 원인은 부분적으로 오프체인(거버넌스/등록 프로세스)이었습니다; 온체인 취약점은 허용 목록에 있는 주소를 신뢰하기 전에 담보 토큰이 진짜인지 검증하지 않은 것이었습니다.*  
  - [https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025](https://www.halborn.com/blog/post/explained-the-ionic-money-hack-february-2025)
  - [https://rekt.news/ionic-money-rekt](https://rekt.news/ionic-money-rekt)

### 모범 사례 및 완화 방법

- 다음을 포함한 **모든 외부 입력을 검증**합니다:
  - 함수 파라미터 (금액, 주소, 구성 값)
  - 오프체인 서명된 데이터 및 콜데이터 페이로드
  - 크로스체인 메시지 및 브리지 페이로드
- **엄격한 불변량 강제**:
  - 수수료, 이율, 레버리지, 담보 비율의 범위.
  - 핵심 주소 및 한도에 대한 0이 아닌 값 요구.
- **커스텀 에러**와 명시적 검사를 사용하여 검증을 명확하고 가스 효율적으로 유지합니다.
- **관리자 및 거버넌스 입력을 검증될 때까지 신뢰할 수 없는 것으로 처리**합니다—잘못된 구성은 명시적 익스플로잇만큼 해로울 수 있습니다.
- 예상치 못한 값이 거부되는지 확인하기 위해 잘못된 입력에 대한 **부정 테스트** (퍼징, 속성 테스트)를 포함합니다.

