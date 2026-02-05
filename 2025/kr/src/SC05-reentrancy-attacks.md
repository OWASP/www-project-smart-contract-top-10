## SC05:2025 - 재진입 공격

### 설명:
재진입 공격은 함수가 자체 상태를 업데이트하기 전에 다른 컨트랙트에 외부 호출을 할 때 스마트 컨트랙트의 취약점을 악용합니다. 이로 인해 악의적일 수 있는 외부 컨트랙트가 원래 함수에 재진입하여 동일한 상태를 사용하여 인출과 같은 특정 작업을 반복할 수 있습니다. 이러한 공격을 통해 공격자는 컨트랙트의 모든 자금을 빼돌릴 수 있습니다.

### 예시 (취약한 컨트랙트): 
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // 취약점: 사용자 잔액을 업데이트하기 전에 이더가 전송되어 재진입이 가능함.
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // 이더 전송 후 잔액 업데이트
        balances[msg.sender] = 0;
    }
}
```
### 영향:
- 가장 즉각적이고 심각한 결과는 자금 고갈입니다. 공격자는 취약점을 악용하여 정당하게 소유한 것보다 더 많은 돈을 인출하며, 잠재적으로 컨트랙트의 잔액을 완전히 비울 수 있습니다.
- 공격자가 무단 함수 호출을 트리거할 수 있습니다. 이로 인해 컨트랙트 또는 관련 시스템 내에서 의도하지 않은 작업이 실행될 수 있습니다.

### 해결 방안:
- 항상 외부 컨트랙트를 호출하기 전에 모든 상태 변경이 이루어지도록 합니다. 즉, 외부 코드를 호출하기 전에 잔액을 업데이트하거나 내부적으로 코드를 수정합니다.
- Open Zeppelin의 Re-entrancy Guard와 같은 재진입을 방지하는 함수 수정자를 사용합니다.

### 예시 (수정된 버전):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // 수정: 이더를 전송하기 전에 사용자 잔액을 업데이트
        balances[msg.sender] = 0;

        // 그 다음 이더 전송
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 재진입 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : 상세 [해킹 분석](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : 상세 [해킹 분석](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)
