## SC06:2026 - 검증되지 않은 외부 호출

#### 설명

검증되지 않은 외부 호출이란 스마트 컨트랙트가 다른 컨트랙트 또는 주소를 (`call`, `delegatecall`, `staticcall`, 또는 `transfer`/`send`와 같은 고수준 호출을 통해) 호출할 때 피호출자의 동작, 반환 값, 또는 재진입 가능성을 **완전히 고려하지 않는** 모든 상황을 말합니다. 호출 컨트랙트는 피호출자가 예상대로 동작할 것이라고 암묵적으로 신뢰합니다—성공을 반환하고, 재진입하지 않으며, 임의의 로직을 실행하지 않을 것이라고. 그 가정이 위반되면 호출자는 일관성 없는 상태에 놓이거나 악용될 수 있습니다.

이는 외부 상호작용을 수행하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi(토큰 전송, DEX 스왑, 볼트 입금, 플래시 론 콜백), NFT(훅이 있는 전송, 마켓플레이스 지불), DAO(제안 콜데이터 실행), 브리지(메시지 릴레이, 자산 전송), 그리고 구성 가능한 프로토콜(임의 콜백, ERC-777/ERC-721/ERC-1155 수신자 훅, ERC-4626 입금/출금 훅). 비EVM 체인에서도 유사한 패턴이 존재합니다 (예: Move의 `vector::borrow`, Solana CPI) 여기서 크로스 프로그램 호출이 재진입하거나 예상치 못하게 동작할 수 있습니다.

주요 점검 영역:

- **토큰 전송** (ERC-20, ERC-721, ERC-1155) 및 비표준 반환 값 또는 되돌리기 동작
- **콜백 및 훅 인터페이스** (ERC-777 `tokensReceived`, ERC-4626 `afterDeposit`/`beforeWithdraw`, `onFlashLoan`, `onERC721Received`)
- **저수준 호출** (`call`, `delegatecall`, `callcode`) 및 가스/스토리지 영향
- **구성 가능성 흐름** (볼트가 전략을 호출하고, 전략이 DEX를 호출하는) 여기서 재진입이 여러 컨트랙트를 가로질러 발생할 수 있음

공격자가 악용하는 방식:

- 전송 시 훅이 있는 콜백 또는 토큰에서 악의적인 로직을 구현하여 **재진입** (SC08 참조)
- 반환 값이 무시될 때(예: 반환하지 않는 ERC-20) **조용한 실패** 및 상태가 일관성 없이 남겨짐
- 사용자 제공 또는 프로토콜 구성 가능한 주소를 호출할 때 **예상치 못한 코드 실행**

검증되지 않은 외부 호출은 *단독 근본 원인*인 경우는 드물지만 재진입(SC08), 비즈니스 로직 익스플로잇(SC02), 회계 불일치의 **중요한 조력자**입니다.

### 예시 (취약한 검증되지 않은 호출 패턴)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract VulnerablePayout {
    IToken public token;

    mapping(address => uint256) public rewards;

    constructor(IToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        require(amount > 0, "no rewards");

        // 취약: 반환 값 또는 재진입 검사 없음
        token.transfer(msg.sender, amount);

        // 외부 호출 후 상태 업데이트
        rewards[msg.sender] = 0;
    }
}
```

**문제점:**

- `transfer` 호출의 반환 값이 무시됩니다; 전송이 실패하면 보상은 0이 아닌 값으로 남지만 사용자는 토큰을 받지 못합니다.
- 상태가 외부 호출 **이후**에 업데이트되어 재진입 가능성이 열립니다 (토큰이 악의적인 경우).

### 예시 (강화된 외부 호출 처리)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ISafeToken {
    function transfer(address to, uint256 amount) external returns (bool);
}

contract SafePayout {
    ISafeToken public immutable token;
    mapping(address => uint256) public rewards;

    error NoRewards();
    error TransferFailed();

    constructor(ISafeToken _token) {
        token = _token;
    }

    function claim() external {
        uint256 amount = rewards[msg.sender];
        if (amount == 0) revert NoRewards();

        // 이 변수에 대한 재진입을 완화하기 위해 외부 호출 *전에* 상태 변경
        rewards[msg.sender] = 0;

        bool ok = token.transfer(msg.sender, amount);
        if (!ok) {
            // 필요한 경우 되돌리고 상태 복원 (간결함을 위해 여기서는 생략)
            revert TransferFailed();
        }
    }
}
```

**보안 개선 사항:**

- 상태가 외부 호출 **전에** 업데이트되어 `rewards`에 대한 단순 재진입을 제한합니다.
- 토큰 전송의 반환 값이 검사됩니다; 실패 시 되돌리기로 조용한 불일치를 방지합니다.

> 참고: 완전한 재진입 보호를 위해서는 SC08을 참조하고 `ReentrancyGuard`, 검사-효과-상호작용, 풀 기반 패턴을 고려하세요.

### 2025년 사례 연구

- **GMX (2025년 7월, $4,200만 손실)**  
  안전하지 않은 외부 상호작용과 외부 호출 후 상태 업데이트로 인해 공격자가 재진입하여 회계를 조작할 수 있었습니다. `executeDecreaseOrder` 함수가 환불 과정에서 공격자가 제공한 컨트랙트 주소로 제어를 이전하여 재진입을 가능하게 했습니다. 외부 호출 순서, 적절한 검사 부재, 피호출자 동작에 대한 가정에 의존이 영향을 증폭시켰습니다.  
  - [https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f](https://blog.solidityscan.com/gmx-v1-hack-analysis-ed0ab0c0dd0f)
  - [https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025](https://www.halborn.com/blog/post/explained-the-gmx-hack-july-2025)

- **Arcadia Finance (2025년 7월, $350만 손실)**  
  `SwapLogic._swapRouter()` 및 `RebalancerSpot` 컨트랙트가 피호출자를 검증하지 않고 `swapData` 파라미터를 통해 사용자 제공 라우터 주소에 **임의의 외부 호출**을 허용했습니다. 공격자는 악의적인 컨트랙트를 라우터와 허용 목록에 있는 ArcadiaAccount 모두로 등록한 다음, 라우터 콜백을 사용하여 특권 실행 컨텍스트를 위장하고 `setAssetManager()` / `flashAction()`을 호출했습니다. 프로토콜은 라우터가 상승된 권한을 갖지 않을 것이라고 가정했지만—이 가정이 코드에서 강제되지 않았습니다. 검증되지 않은 콜백과 외부 피호출자 동작에 대한 신뢰가 소진을 가능하게 했습니다.  
  - [https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d](https://blog.solidityscan.com/arcadia-finance-hack-analysis-a03a722e554d)
  - [https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025](https://www.guardrail.ai/blog/arcadia-finance-hack-july-2025)

### 모범 사례 및 완화 방법

- **모든 외부 호출을 신뢰할 수 없는 것으로 처리**:
  - "표준" 토큰이나 잘 알려진 프로토콜도 업그레이드되거나 교체될 수 있습니다.
  - 재진입하거나 예상치 못하게 되돌릴 수 있다고 가정합니다.
- **검사-효과-상호작용** 패턴 사용:
  - 사전 조건 검증.
  - 내부 상태 업데이트.
  - 그런 다음에만 외부 호출 수행.
- 지불에 **풀** 방식을 **푸시** 방식보다 선호:
  - 루프에서 임의의 주소로 자금을 푸시하는 대신 사용자가 출금할 수 있도록 허용합니다.
- 반환 값을 검사하고 모든 실패 모드를 처리합니다:
  - 토큰 작업을 래핑하기 위해 OpenZeppelin의 `SafeERC20`과 같은 라이브러리를 사용합니다.
- 다음에 각별히 주의합니다:
  - 저수준 호출 (`call`, `delegatecall`, `callcode`)
  - 임의 콜백 (예: 훅, onERC721Received, onFlashLoan 콜백)

