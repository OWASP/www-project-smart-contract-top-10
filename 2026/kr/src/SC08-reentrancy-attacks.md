## SC08:2026 - 재진입 공격

#### 설명

재진입이란 스마트 컨트랙트가 외부 호출(다른 컨트랙트 또는 주소로)을 수행하고, 피호출자가 첫 번째 호출이 완료되고 상태가 완전히 업데이트되기 전에 **원래 컨트랙트로 다시 호출**할 수 있는 모든 상황을 말합니다. 호출자가 재진입 안전하게 설계되지 않은 경우, 콜백은 오래된 상태를 관찰하고 이를 악용할 수 있습니다—예를 들어, 호출자의 잔액보다 더 많이 출금하거나, 보상을 이중으로 계산하거나, 복잡한 다단계 흐름에서 회계를 조작합니다.

이는 외부 호출을 수행하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi(토큰 전송, DEX 스왑, 볼트 입금/출금, 플래시 론 콜백), NFT(ERC-721/ERC-1155 수신자 훅이 있는 전송, 마켓플레이스 지불), DAO(외부 컨트랙트를 호출하는 제안 실행), 브리지(메시지 릴레이, 자산 전송), 그리고 구성 가능한 프로토콜(ERC-777 훅, ERC-4626 입금/출금 훅). 재진입은 **단일 함수**(동일 함수의 재귀 호출), **크로스 함수**(다른 함수로의 콜백), 또는 **크로스 컨트랙트**(콜백이 여러 컨트랙트를 가로지름)일 수 있습니다. 비EVM 체인에서도 크로스 프로그램 호출이 재귀할 수 있는 곳이라면 유사한 패턴이 존재합니다.

주요 점검 영역:

- **클래식 출금-전-업데이트** (외부 호출 후 상태 변경)
- **콜백 및 훅 인터페이스** (ERC-777, ERC-721/1155 수신자, ERC-4626, 플래시 론 콜백)
- **크로스 함수 재진입** (함수 간 읽기-쓰기 가정)
- **읽기 전용 재진입** (콜백 중 상태를 읽는 뷰 함수 또는 오라클)
- **크로스 컨트랙트 및 멀티 모듈** 재진입 (볼트 → 전략 → DEX 흐름)

공격자가 악용하는 방식:

- 전송/훅 콜백 시 재진입하는 **악의적인 토큰 또는 수신자**
- 상환 전에 공격자 로직을 실행하는 **플래시 론 콜백**
- 트랜잭션 중간에 모듈 전반에 걸쳐 상태가 일관성 없는 **복잡한 호출 그래프**

### 예시 (취약한 재진입 패턴)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerableVault {
    IToken public immutable token;
    mapping(address => uint256) public balances;

    constructor(IToken _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // 간결함을 위해 토큰이 이미 전송되었다고 가정
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external {
        require(balances[msg.sender] >= amount, "insufficient");

        // 상태 업데이트 전 외부 호출 – 재진입 창
        token.transfer(msg.sender, amount);

        balances[msg.sender] -= amount;
    }
}
```

**문제점:**

- 악의적인 토큰/프록시를 사용하는 공격자는 `transfer` 호출 내에서 `withdraw`로 재진입하여, 변경되지 않은 `balances[msg.sender]`를 기반으로 반복적으로 출금할 수 있습니다.

### 예시 (재진입 안전한 출금)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface ITokenSafe {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafeVault is ReentrancyGuard {
    ITokenSafe public immutable token;
    mapping(address => uint256) public balances;

    error InsufficientBalance();
    error TransferFailed();

    constructor(ITokenSafe _token) {
        token = _token;
    }

    function deposit(uint256 amount) external {
        // 간결함을 위해 토큰이 이미 전송되었다고 가정
        balances[msg.sender] += amount;
    }

    function withdraw(uint256 amount) external nonReentrant {
        uint256 bal = balances[msg.sender];
        if (bal < amount) revert InsufficientBalance();

        // 효과 먼저
        balances[msg.sender] = bal - amount;

        // 그 다음 상호작용
        bool ok = token.transfer(msg.sender, amount);
        if (!ok) revert TransferFailed();
    }
}
```

**보안 개선 사항:**

- OpenZeppelin의 `ReentrancyGuard`에서 `nonReentrant` 수정자 사용.
- 재진입 창을 최소화하기 위해 **검사-효과-상호작용** 패턴 적용.
- 전송 반환 값을 검사하고 실패 시 되돌립니다.

### 2025년 사례 연구

- **GMX (2025년 7월, $4,200만 손실)**  
  GMX V1 컨트랙트가 `executeDecreaseOrder`의 클래식하지만 정교한 재진입 벡터를 통해 익스플로잇되었습니다. 이 함수는 공격자의 스마트 컨트랙트 주소를 파라미터로 수락했습니다; 환불 과정에서 해당 주소로 제어를 이전할 때, 공격자는 재진입하여 글로벌 평균 숏 가격, AUM, GLP 가치를 조작했습니다. 외부 호출 후 상태 업데이트와 재진입 가드 부재가 소진을 가능하게 했습니다. 이 취약점은 2022년에 감사되지 않은 패치로 도입되었습니다.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)
  - [https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis](https://blog.verichains.io/p/gmx-42m-exploit-root-cause-analysis)

### 모범 사례 및 완화 방법

- 다음과 같은 상태 저장 함수에 **`ReentrancyGuard`** 또는 유사한 재진입 잠금 사용:
  - 잔액 또는 내부 회계를 수정하는 함수.
  - 컨트랙트로 재진입할 수 있는 외부 호출을 수행하는 함수.
- **검사-효과-상호작용** 따르기:
  - 사전 조건 검사.
  - 모든 상태 변경 적용.
  - 그런 다음에만 외부 컨트랙트 호출.
- **ERC-777 훅, ERC-4626 훅, 기타 콜백**을 재진입 벡터로 취급합니다.
- 다음을 신중하게 검토합니다:
  - 크로스 함수 상호작용 (예: `deposit`이 내부적으로 `withdraw`를 호출하는 경우).
  - 재진입이 단일 컨트랙트 내가 아닌 모듈 전반에 걸쳐 발생할 수 있는 멀티 컨트랙트 시스템.
- 외부 호출이 포함된 경우 특히 재진입 중심의 **퍼징 및 단위 테스트** 포함.

