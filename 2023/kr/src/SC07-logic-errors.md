## 취약점: 로직 오류

### 설명:
로직 오류는 비즈니스 로직 취약점이라고도 하며, 스마트 컨트랙트의 미묘한 결함입니다. 컨트랙트의 코드가 의도한 동작과 일치하지 않을 때 발생합니다. 이러한 오류는 찾기 어렵고, 컨트랙트의 로직 내에 숨어서 발견되기를 기다립니다.

### 예시:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LendingPlatform {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");
        
        // 결함 있는 계산: 총 대출 풀을 업데이트하지 않고 사용자 잔액을 잘못 감소시킴
        userBalances[msg.sender] -= amount;
        
        // 여기서 총 대출 풀을 업데이트해야 하지만 생략됨
        
        payable(msg.sender).transfer(amount);
    }
}
```
### 영향:
- 로직 오류는 스마트 컨트랙트가 예상치 못하게 동작하거나 심지어 완전히 사용할 수 없게 만들 수 있습니다. 이러한 오류는 자금 손실, 토큰의 잘못된 분배 또는 기타 부정적인 결과를 초래할 수 있으며, 사용자와 이해관계자에게 심각한 재정적 및 운영적 결과를 초래할 수 있습니다.

### 해결 방안:
- 가능한 모든 비즈니스 로직을 다루는 포괄적인 테스트 케이스를 작성하여 항상 코드를 검증하세요.
- 철저한 코드 리뷰와 감사를 수행하여 잠재적인 로직 오류를 식별하고 수정하세요.
- 각 함수와 모듈의 의도된 동작을 문서화한 다음 실제 구현과 비교하여 일치하는지 확인하세요.

### 비즈니스 로직 공격 피해 사례:
1. [Level Finance 해킹](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : 상세한 [해킹 분석](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO 해킹](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)
