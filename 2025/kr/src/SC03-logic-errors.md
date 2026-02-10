## SC03:2025 - 로직 오류

### 설명: 
로직 오류는 비즈니스 로직 취약점이라고도 하며, 스마트 컨트랙트의 미묘한 결함입니다. 컨트랙트의 코드가 의도된 동작과 일치하지 않을 때 발생합니다. 이러한 오류는 보상 분배의 잘못된 계산, 부적절한 토큰 발행 메커니즘, 또는 대출 및 차용 로직의 잘못된 계산과 같은 다양한 형태로 나타날 수 있습니다. 이러한 취약점은 컨트랙트의 로직 내에 숨어 발견되기를 기다리며 파악하기 어렵습니다.

#### 로직 오류의 예시:
1. **잘못된 보상 분배:** 이해관계자들 사이의 보상 분배 계산 오류로 인한 불공정한 할당.
2. **부적절한 토큰 발행:** 무한하거나 의도하지 않은 토큰 생성을 허용하는 검증되지 않거나 잘못된 발행 로직.
3. **대출 풀 불균형:** 입금과 출금의 잘못된 추적으로 인한 풀 준비금의 불일치.

### 예시 (취약한 컨트랙트):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");

        // 잘못된 계산: 총 대출 풀을 업데이트하지 않고 사용자 잔액만 잘못 감소시킴
        userBalances[msg.sender] -= amount;

        // 총 대출 풀을 업데이트해야 하지만 여기서는 누락됨.

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        // 잘못된 발행 로직: 보상 금액이 검증되지 않음
        userBalances[to] += rewardAmount;
    }
}
```

### 영향:
- 로직 오류는 스마트 컨트랙트가 예기치 않게 동작하거나 완전히 사용 불가능하게 만들 수 있습니다. 이러한 오류는 다음과 같은 결과를 초래할 수 있습니다:
  - **자금 손실:** 잘못된 보상 분배 또는 풀 불균형으로 인한 컨트랙트 자금 고갈.
  - **과도한 토큰 발행:** 토큰 공급량 인플레이션으로 신뢰와 가치 훼손.
  - **운영 실패:** 컨트랙트가 의도된 기능을 수행하지 못함.
- 이러한 결과는 사용자와 이해관계자에게 심각한 재정적 및 운영적 손실을 초래할 수 있습니다.

### 해결 방안:
- 모든 가능한 비즈니스 로직 시나리오를 다루는 포괄적인 테스트 케이스를 작성하여 항상 코드를 검증합니다.
- 잠재적인 로직 오류를 식별하고 수정하기 위해 철저한 코드 리뷰와 감사를 수행합니다.
- 각 함수와 모듈의 의도된 동작을 문서화하고 실제 구현과 비교하여 일치하는지 확인합니다.
- 다음과 같은 가드레일을 구현합니다:
  - 계산 오류를 방지하기 위한 안전한 수학 라이브러리.
  - 토큰 발행에 대한 적절한 검사와 균형.
  - 감사 가능한 보상 분배 알고리즘.

### 예시 (수정된 버전):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");

        // 사용자 잔액과 총 대출 풀을 올바르게 감소시킴
        userBalances[msg.sender] -= amount;
        totalLendingPool -= amount;

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        require(rewardAmount > 0, "Reward amount must be positive");

        // 보호된 발행 로직
        userBalances[to] += rewardAmount;
    }
}
```

### 비즈니스 로직 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Level Finance 해킹](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : 상세 [해킹 분석](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO 해킹](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : 상세 [해킹 분석](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)
