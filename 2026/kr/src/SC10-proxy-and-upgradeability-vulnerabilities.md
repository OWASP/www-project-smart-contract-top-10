## SC10:2026 - 프록시 및 업그레이드 가능성 취약점

#### 설명

프록시 및 업그레이드 가능성 취약점이란 스마트 컨트랙트가 업그레이드 가능한 아키텍처(프록시, 비콘, 또는 구현체 교체 패턴)를 사용하고 업그레이드 경로, 초기화, 또는 관리자 제어가 잘못 설계되거나 잘못 구성된 모든 상황을 말합니다. 업그레이드 가능한 컨트랙트는 **프록시**(상태를 보유하고 호출을 위임)와 **구현체**(로직을 포함)를 분리합니다. 업그레이드 가능성이 제대로 보안되지 않으면, 공격자는 프록시 어드민 또는 업그레이드 역할을 탈취하여 악의적인 구현체를 배포하거나, 소유권을 탈취하기 위해 컨트랙트를 재초기화하거나, 초기화 또는 마이그레이션 단계에서 중요한 검사를 우회할 수 있습니다.

이는 업그레이드 가능성을 사용하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi(대출, 볼트, DEX), NFT(컬렉션, 마켓플레이스), DAO(거버넌스, 트레저리), 브리지(메신저, 자산 컨트랙트), L2/크로스체인 시스템. 일반적인 패턴으로는 투명 프록시, UUPS(EIP-1822), 비콘 프록시, 커스텀 라우터-구현체 설계가 있습니다. 비EVM 체인에서도 유사한 업그레이드 메커니즘이 존재합니다 (예: Move 모듈, Solana 프로그램 업그레이드) 유사한 신뢰 및 초기화 위험이 있습니다.

주요 점검 영역:

- **업그레이드 및 관리자 역할** (구현체를 변경할 수 있는 주체, 스토리지 레이아웃 호환성)
- **초기화 및 재초기화** (보호되지 않은 `initialize`, 누락된 `initializer` 가드, 스토리지 충돌)
- **프록시 위임** (`delegatecall` 컨텍스트, `msg.sender`/`msg.value` 전파)
- **스토리지 레이아웃** (프록시와 구현체 간 슬롯 충돌, 추가 전용 스토리지)
- **타임락 및 거버넌스** (업그레이드 프로세스, 롤백 기능)

공격자가 악용하는 방식:

- 모든 호출자가 프록시를 악의적인 구현체로 가리킬 수 있는 **보호되지 않은 업그레이드 함수**
- 소유권, 구성, 또는 접근 제어를 재설정하는 **재초기화**
- 공격자 제어 파라미터로 구현체가 초기화될 수 있는 **delegatecall을 통한 초기화**
- 덮어쓰기로 이어지는 프록시와 구현체 간 **스토리지 충돌**

이러한 문제는 종종 접근 제어(SC01)와 겹치지만 프록시 및 업그레이드 메커니즘의 시스템적 영향으로 인해 별도의 주의가 필요합니다.

### 예시 (취약한 업그레이드 가능한 프록시 어드민)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableProxyAdmin {
    address public admin;
    address public implementation;

    constructor(address _implementation) {
        // 중요: 커스텀 어드민을 설정할 방법이 없음; 암묵적으로 배포자 로직을 신뢰
        admin = msg.sender;
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external {
        // 누락: 접근 제어 (어드민만) 및 건전성 검사
        implementation = newImplementation;
    }
}
```

**문제점:**

- `upgrade`에 접근 제어 없음; 누구나 `implementation`을 변경할 수 있습니다.
- `newImplementation`에 대한 검사 없음 (예: 인터페이스 호환성, 0이 아닌 주소).

### 예시 (더 안전한 업그레이드 가능성 패턴)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract SafeProxyAdmin is Ownable {
    address public implementation;

    event Upgraded(address indexed newImplementation);

    error InvalidImplementation();

    constructor(address _implementation) {
        _setImplementation(_implementation);
        _transferOwnership(msg.sender);
    }

    function _setImplementation(address _implementation) internal {
        if (_implementation == address(0)) revert InvalidImplementation();
        implementation = _implementation;
    }

    function upgrade(address newImplementation) external onlyOwner {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }
}
```

**보안 개선 사항:**

- `upgrade`가 컨트랙트 소유자로 제한됩니다 (이 자체가 견고한 거버넌스 또는 멀티시그여야 합니다).
- 구현체 주소가 검증되고 업그레이드가 이벤트를 통해 기록됩니다.

### 초기화 및 재초기화 위험

초기화 함수(예: `initialize()`, OpenZeppelin의 `initializer` 수정자)는 업그레이드 가능한 패턴에서 중요합니다. 일반적인 함정:

- 누구나 호출할 수 있는 **보호되지 않은 초기화자**.
- 소유권, 구성, 또는 상태를 재설정할 수 있는 **재초기화**.
- 의도치 않은 방식으로 프록시에서 **delegatecall을 통해 도달할 수 있는** 초기화 로직.

기본 예시:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract VulnerableLogic {
    address public owner;

    // 초기화자 가드 누락
    function initialize(address _owner) external {
        owner = _owner;
    }
}
```

적절한 초기화 제어 없이 프록시 뒤에서 사용되면, 공격자는 프록시를 통해 `initialize`를 호출하여 자신을 소유자로 설정하고 프로토콜을 탈취할 수 있습니다.

### 2025년 사례 연구

- **Kinto Protocol (2025년 7월, $155만 손실)**  
  공격자가 **초기화되지 않은 ERC1967 프록시 컨트랙트**를 악용했습니다. 그들은 제대로 초기화되지 않은 새로 배포된 프록시 컨트랙트를 감지한 다음, 잠복 백도어가 포함된 악의적인 구현체로 초기화했습니다. 몇 달 후, 공격자는 백도어를 활성화하고 프록시를 악의적인 코드로 업그레이드하여 K 토큰을 직접 발행해 $155만을 소진했습니다. 취약점: 누구나 프록시 어드민이 될 수 있는 **보호되지 않은 초기화**.  

- **초기화되지 않은 프록시 캠페인 (2025년, 여러 프로토콜에 걸쳐 $1,000만+ 손실)**  
  더 광범위한 캠페인이 여러 EVM 체인의 초기화되지 않은 ERC1967 프록시를 표적으로 삼았습니다. 공격자는 자동화된 스캐닝을 사용하여 합법적인 개발자가 초기화하기 전에 새로 배포된 프록시를 감지한 다음, 악의적인 구현체로 초기화했습니다. 백도어는 몇 달 동안 잠복하여 감사를 회피했습니다. 활성화되면 공격자는 프록시를 업그레이드하고 자금을 소진할 수 있었습니다.  
  - [https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses](https://audita.io/blog-articles/the-proxy-hack-uninitialized-contracts-costing-defi-10m-in-losses)
  - [https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6](https://medium.com/mamori-finance/post-mortem-k-proxy-hack-our-path-forward-c2c3809882c6)

### 모범 사례 및 완화 방법

- 맞춤형 설계 대신 **잘 확립된 프록시 패턴 및 라이브러리** 사용 (예: OpenZeppelin UUPS/투명 프록시).
- 업그레이드 및 관리자 역할을 **견고한 거버넌스 / 멀티시그**로 보호합니다; 강력한 운영 제어 없이 EOA에 남겨두지 않습니다.
- **초기화자 가드 적용**:
  - `initializer` 및 `reinitializer` 수정자를 올바르게 사용합니다.
  - 직접 초기화를 방지하기 위해 배포 후 구현체 컨트랙트를 잠급니다.
- 업그레이드에 **타임락 및 다단계 프로세스 요구**:
  - 업그레이드 제안을 공지합니다.
  - 실행 전에 검토/모니터링 시간을 허용합니다.
- 다음을 포함한 포괄적인 **업그레이드 런북 및 체크리스트** 유지:
  - 마이그레이션 테스트.
  - 새 구현체 코드 및 스토리지 레이아웃 검증.
  - 가능한 경우 업그레이드 단계의 온체인 시뮬레이션.

