## 취약점: 검증되지 않은 외부 호출

### 설명:
검증되지 않은 외부 호출은 컨트랙트가 해당 호출의 결과를 적절하게 확인하지 않고 다른 컨트랙트나 주소에 외부 호출을 할 때 발생하는 보안 결함을 말합니다. 이더리움에서 컨트랙트가 다른 컨트랙트를 호출할 때, 호출된 컨트랙트는 예외를 던지지 않고 조용히 실패할 수 있습니다. 호출하는 컨트랙트가 반환 값을 확인하지 않으면, 실제로는 실패했더라도 호출이 성공했다고 잘못 가정할 수 있습니다. 이는 컨트랙트 상태의 불일치와 공격자가 악용할 수 있는 취약점으로 이어질 수 있습니다.

### 예시:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.24;

contract Proxy {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    function forward(address callee, bytes _data) public {
        require(callee.delegatecall(_data));
    }
}
```
### 영향:
- 검증되지 않은 외부 호출은 트랜잭션 실패를 초래하여 의도한 작업이 성공적으로 완료되지 않을 수 있습니다. 이는 컨트랙트가 전송이 성공했다는 잘못된 가정하에 진행될 수 있으므로 자금 손실로 이어질 수 있습니다. 또한 잘못된 컨트랙트 상태를 만들어 컨트랙트를 추가 공격과 로직의 불일치에 취약하게 만들 수 있습니다.

### 해결 방안:
- 가능하면 send() 대신 transfer()를 사용하세요. transfer()는 외부 호출이 실패하면 트랜잭션을 되돌립니다.
- 적절한 처리를 보장하기 위해 항상 send() 또는 call() 함수의 반환 값을 확인하여 false를 반환하는 경우를 처리하세요.

### 검증되지 않은 외부 호출 공격 피해 사례:
1. [Punk Protocol 해킹](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : 상세한 [해킹 분석](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)
