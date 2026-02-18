## SC01:2026 - 접근 제어 취약점

#### 설명

부적절한 접근 제어란 스마트 컨트랙트가 *누가* 특권 기능을 호출할 수 있는지, *어떤 조건* 하에서, *어떤 파라미터*로 호출할 수 있는지를 엄격하게 강제하지 않는 모든 상황을 말합니다. 현대 DeFi 시스템에서 이는 단순한 `onlyOwner` 수정자를 훨씬 넘어섭니다. 거버넌스 컨트랙트, 멀티시그, 가디언, 프록시 어드민, 크로스체인 라우터 모두 토큰 발행/소각, 준비금 이동, 풀 재구성, 핵심 로직 일시정지/재개, 구현체 업그레이드 등의 권한을 가진 주체가 누구인지 강제하는 데 참여합니다. 이러한 신뢰 경계 중 하나라도 취약하거나 일관성 없이 적용된다면, 공격자는 권한 있는 주체를 사칭하거나 신뢰할 수 없는 주소를 권한 있는 것처럼 시스템이 처리하도록 만들 수 있습니다.

주요 점검 영역:
- **소유권 / 관리자 제어** (예: `onlyOwner`, 거버넌스, 멀티시그)
- **업그레이드 및 일시정지 메커니즘** (프록시 어드민, 가디언)
- **자금 이동 및 회계** (발행/소각, 풀 재구성, 수수료 라우팅)
- **크로스체인 또는 크로스모듈 신뢰 경계** (브리지, 볼트 라우터, L2 메신저)

공격자가 악용하는 방식:

- 민감한 함수에 **수정자 또는 역할 검사 누락**
- **`msg.sender`에 대한 잘못된 가정** (예: 델리게이트 콜 또는 메타 트랜잭션을 통한 경우)
- 컨트랙트 또는 프록시의 **보호되지 않은 초기화 / 재초기화**
- 모듈 간 **권한 혼동** (예: 오프바이원 검사, 잘못된 신뢰 주소)

다른 문제(예: 로직 버그, 업그레이드 가능성 결함)와 결합되면 접근 제어 실패는 프로토콜 전체 침해로 이어질 수 있습니다.

### 예시 (취약한 컨트랙트)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LiquidityPoolVulnerable {
    address public owner;
    mapping(address => uint256) public balances;

    constructor() {
        owner = msg.sender;
    }

    // 누구나 새 소유자를 설정할 수 있음 – 치명적인 접근 제어 버그
    function transferOwnership(address newOwner) external {
        owner = newOwner; // 접근 제어 없음
    }

    // 소유자만 호출하여 토큰을 구출하도록 의도됨
    function emergencyWithdraw(address to, uint256 amount) external {
        // 누락: require(msg.sender == owner)
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;
    }
}
```

**문제점:**

- `transferOwnership`은 누구나 호출 가능하여 임의적인 탈취를 허용합니다.
- `emergencyWithdraw`에 접근 제어가 전혀 없어, 사실상 누구든 컨트랙트 잔액을 소진할 수 있습니다.

### 예시 (역할 기반 접근 제어가 적용된 수정 버전)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/AccessControl.sol";

contract LiquidityPoolSecure is AccessControl {
    bytes32 public constant GOVERNANCE_ROLE = keccak256("GOVERNANCE_ROLE");
    bytes32 public constant GUARDIAN_ROLE = keccak256("GUARDIAN_ROLE");

    mapping(address => uint256) public balances;

    event EmergencyWithdraw(address indexed to, uint256 amount, address indexed triggeredBy);

    constructor(address governance, address guardian) {
        _grantRole(DEFAULT_ADMIN_ROLE, governance);
        _grantRole(GOVERNANCE_ROLE, governance);
        _grantRole(GUARDIAN_ROLE, guardian);
    }

    function grantGovernance(address newGov) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _grantRole(GOVERNANCE_ROLE, newGov);
    }

    function setGuardian(address newGuardian) external onlyRole(GOVERNANCE_ROLE) {
        _grantRole(GUARDIAN_ROLE, newGuardian);
    }

    // 거버넌스 또는 지정된 가디언만 긴급 출금을 실행할 수 있음
    function emergencyWithdraw(address to, uint256 amount)
        external
        onlyRole(GUARDIAN_ROLE)
    {
        require(to != address(0), "invalid to");
        require(balances[address(this)] >= amount, "insufficient");
        balances[address(this)] -= amount;
        balances[to] += amount;

        emit EmergencyWithdraw(to, amount, msg.sender);
    }
}
```

**보안 개선 사항:**

- 명시적인 **RBAC**: `DEFAULT_ADMIN_ROLE`, `GOVERNANCE_ROLE`, `GUARDIAN_ROLE`.
- 신뢰할 수 있는 역할만 권한 재구성 또는 긴급 출금을 수행할 수 있습니다.
- 구성(거버넌스)과 긴급 대응(가디언) 간의 명확한 분리.

### 2025년 사례 연구

- **Balancer V2 (2025년 11월, 약 $1억 2,800만 손실)**  
  복잡한 멀티체인 풀 생태계가 풀 구성 및 소유권 가정의 결함 있는 접근 제어로 인해 피해를 입었습니다. `manageUserBalance` 함수에 부적절한 접근 제어가 있었습니다. 이 함수는 `msg.sender`를 사용자가 제공한 `op.sender` 값과 비교했는데, 공격자는 이를 `msg.sender`와 일치하도록 설정하여 보호를 우회하고 풀 컨트롤러로 위장해 무단 WITHDRAW_INTERNAL 작업을 실행할 수 있었습니다. 이는 `_upscaleArray`의 반올림 오류와 연계되어 유동성을 소진시켰습니다.  
  핵심 교훈:
  - 중요한 풀 작업은 **명시적인 역할 검사**와 온체인 거버넌스로 보호되어야 합니다.
  - 크로스체인 또는 크로스모듈 "소유자" 개념은 메시지 출처에서 가정하지 않고 **온체인에서 검증**되어야 합니다.  
  - [https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit](https://www.openzeppelin.com/news/understanding-the-balancer-v2-exploit)
  - [https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/](https://research.checkpoint.com/2025/how-an-attacker-drained-128m-from-balancer-through-rounding-error-exploitation/)
  - [https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025](https://www.halborn.com/blog/post/explained-the-balancer-hack-november-2025)

- **Zoth (2025년 3월, $840만 손실)**  
  핵심 회계 및 관리 기능 주변의 부적절한 권한 검사로 인해 공격자가 무단 자금 이동을 수행할 수 있었습니다. 공격자는 Zoth의 배포자 지갑(관리자를 제어하는 단일 EOA)을 탈취하고 USD0PPSubVaultUpgradeable 프록시에 악의적인 업그레이드를 수행하여 악성 구현체를 배포해 $840만을 인출했습니다. 프로토콜은 취약한 가정에 의존했습니다—단일 개인 키가 중요한 관리자 기능을 제어하는 구조였습니다.  
  핵심 교훈:
  - 주소에 대한 **암묵적 신뢰** 회피 (예: "배포자는 영원히 신뢰됨").
  - 운영, 긴급, 업그레이드 역할 간 명확한 분리를 갖춘 **역할 기반 접근 제어(RBAC)** 사용.  
  - [https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b](https://blog.solidityscan.com/zoth-hack-analysis-80ba3ac5076b)
  - [https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025](https://www.halborn.com/blog/post/explained-the-zoth-hack-march-2025)

- **Cork Protocol (2025년 5월, $1,100~1,200만 손실)**  
  Uniswap V4 훅 콜백(예: `beforeSwap`)에 적절한 접근 제어가 없었습니다. 호출자가 신뢰할 수 있는 PoolManager인지 검증하지 않았습니다. `beforeSwap` 함수에 `onlyPoolManager` 수정자가 없었습니다. 공격자는 임의의 파라미터로 훅을 직접 호출하여 프로토콜이 파생 토큰을 자신에게 적립하도록 속였습니다. 근본 원인은 훅 진입점에서 **호출자 검증 누락**이었습니다.  
  핵심 교훈:
  - 훅 및 콜백 진입점은 온체인에서 명시적으로 **호출자를 검증**해야 합니다 (예: onlyPoolManager).  
  - [https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/](https://dedaub.com/blog/the-11m-cork-protocol-hack-a-critical-lesson-in-uniswap-v4-hook-security/)
  - [https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit](https://www.coindesk.com/business/2025/05/28/a16z-backed-cork-protocol-suffers-usd12m-smart-contract-exploit)
  - [https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cork-protocol-hack-may-2025)

### 모범 사례 및 완화 방법

견고한 접근 제어는 맞춤형 역할 시스템 대신 OpenZeppelin의 `Ownable` 및 `AccessControl`과 같이 검증된 기본 요소를 사용하는 것에서 시작합니다. 특권 역할은 적게 유지하고, 명확히 문서화하며, EOA 대신 잘 보안된 멀티시그 또는 거버넌스 모듈이 보유하는 것이 이상적입니다. 업그레이드 가능한 컨트랙트의 초기화 루틴은 첫 사용 후 잠겨야 하며, 재초기화 공격을 방지하기 위해 `initializer`/`reinitializer` 가드와 명시적 버전 관리가 필요합니다. 프록시 및 핵심 구성 요소의 업그레이드 경로는 엄격하게 제어되고 관찰 가능해야 하며, 오프체인 모니터링이 남용을 신속하게 감지할 수 있도록 모든 권한 변경 또는 업그레이드에 대해 이벤트를 발생시켜야 합니다. 마지막으로, 접근 제어 정책은 테스트, 퍼징 속성, 가능한 경우 공식 명세에 인코딩되어야 하며, "권한 없는 주소는 자금을 소진하거나 관리자 제어를 탈취할 수 없다"와 같은 속성을 검증해야 합니다.

