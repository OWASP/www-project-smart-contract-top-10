# SC04:2025 - 입력 검증 부재

## 설명:
입력 검증은 스마트 컨트랙트가 유효하고 예상된 데이터만 처리하도록 보장합니다. 컨트랙트가 들어오는 입력을 검증하지 못하면, 로직 조작, 무단 접근, 예기치 않은 동작과 같은 보안 위험에 의도치 않게 노출됩니다. 예를 들어, 컨트랙트가 검증 없이 사용자 입력이 항상 유효하다고 가정하면, 공격자는 이 신뢰를 악용하여 악성 데이터를 주입할 수 있습니다. 이러한 입력 검증 부재는 스마트 컨트랙트의 보안과 신뢰성을 손상시킵니다.

## 예시 (취약한 컨트랙트):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LackOfInputValidation {
    mapping(address => uint256) public balances;

    function setBalance(address user, uint256 amount) public {
        // 이 함수는 검증 없이 누구나 모든 사용자의 임의 잔액을 설정할 수 있게 허용함.
        balances[user] = amount;
    }
}
```
### 영향:
- 공격자가 입력을 조작하여 자금을 빼돌리거나, 토큰을 훔치거나, 기타 재정적 피해를 입힐 수 있습니다.
- 부적절한 입력은 상태 변수를 손상시켜 신뢰할 수 없고 안전하지 않은 컨트랙트 동작을 초래할 수 있습니다.
- 공격자가 컨트랙트를 악용하여 무단 거래나 작업을 수행하여 사용자와 전체 시스템 모두에 영향을 미칠 수 있습니다.

### 해결 방안:
- 입력이 예상 타입에 부합하는지 확인합니다.
- 입력이 허용 가능한 범위 내에 있는지 검증합니다.
- 권한이 있는 엔티티만 특정 함수를 호출할 수 있도록 보장합니다.
- 주소 형식이나 문자열 길이와 같은 입력의 구조를 검증합니다.
- 입력이 검증에 실패하면 항상 실행을 중단하고 명확한 오류 메시지를 제공합니다.

### 예시 (수정된 버전):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LackOfInputValidation {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not authorized");
        _;
    }

    function setBalance(address user, uint256 amount) public onlyOwner {
        require(user != address(0), "Invalid address");
        balances[user] = amount;
    }
}
```
### 입력 검증 부재로 인한 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Convergence Finance](https://etherscan.io/address/0x2b083beaaC310CC5E190B1d2507038CcB03E7606#code) : 상세 [해킹 분석](https://blog.solidityscan.com/convergence-finance-hack-analysis-12e6acd9ea08)
2. [Socket Gateway](https://etherscan.io/address/0x3a23F943181408EAC424116Af7b7790c94Cb97a5#code) : 상세 [해킹 분석](https://blog.solidityscan.com/socket-gateway-hack-analysis-b0e9567f7d3e)
