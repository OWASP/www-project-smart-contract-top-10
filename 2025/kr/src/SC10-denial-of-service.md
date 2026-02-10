## SC10:2025 - 서비스 거부 공격

### 설명:
Solidity에서의 서비스 거부(DoS) 공격은 가스, CPU 사이클, 스토리지와 같은 리소스를 고갈시키는 취약점을 악용하여 스마트 컨트랙트를 사용 불가능하게 만드는 것을 포함합니다. 일반적인 유형으로는 악의적인 행위자가 과도한 가스를 필요로 하는 트랜잭션을 생성하는 가스 고갈 공격, 컨트랙트 호출 시퀀스를 악용하여 무단으로 자금에 접근하는 재진입 공격, 블록 가스를 소비하여 정당한 트랜잭션을 방해하는 블록 가스 한도 공격이 있습니다.

### 예시 (취약한 컨트랙트):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        // 현재 왕이 되돌리는 악의적인 폴백 함수를 가지고 있으면, 새로운 왕이 왕좌를 차지하는 것을 방지하여 서비스 거부를 일으킵니다.
        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```
### 영향:
- 성공적인 DoS 공격은 스마트 컨트랙트를 응답하지 않게 만들어 사용자가 의도된 대로 상호 작용하는 것을 방지할 수 있습니다. 이는 컨트랙트에 의존하는 중요한 운영과 서비스를 방해할 수 있습니다.
- DoS 공격은 특히 스마트 컨트랙트가 자금이나 자산을 관리하는 탈중앙화 애플리케이션(dApps)에서 재정적 손실을 초래할 수 있습니다.
- DoS 공격은 스마트 컨트랙트와 관련 플랫폼의 평판을 손상시킬 수 있습니다. 사용자는 플랫폼의 보안과 신뢰성에 대한 신뢰를 잃어 사용자와 비즈니스 기회의 손실로 이어질 수 있습니다.
  
### 해결 방안:
- 스마트 컨트랙트가 잠재적으로 실패할 수 있는 외부 호출의 비동기 처리와 같이 일관된 실패를 처리할 수 있도록 하여 컨트랙트 무결성을 유지하고 예기치 않은 동작을 방지합니다.
- 외부 호출, 루프, 순회에 `call`을 사용할 때 과도한 가스 소비를 피하도록 주의합니다. 이는 트랜잭션 실패나 예기치 않은 비용으로 이어질 수 있습니다.
- 컨트랙트 권한에서 단일 역할에 과도한 권한을 부여하지 않습니다. 대신 권한을 합리적으로 분할하고 개인 키 침해로 인한 권한 손실을 방지하기 위해 중요한 권한을 가진 역할에 대해 다중 서명 지갑 관리를 사용합니다.

### 예시 (수정된 버전):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    // transfer와 같이 고정 가스 비용을 가진 더 안전한 방법을 사용하여 자금을 전송합니다.
    // 이는 call 사용을 피하고 악의적인 폴백 함수 문제를 방지합니다.
    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        address previousKing = king;
        uint256 previousBalance = balance;

        // 재진입 문제를 방지하기 위해 이더를 전송하기 전에 상태를 업데이트합니다.
        king = msg.sender;
        balance = msg.value;

        // 악의적인 폴백으로 인해 트랜잭션이 실패하지 않도록 call 대신 transfer를 사용합니다.
        payable(previousKing).transfer(previousBalance);
    }
}
```
