## SC06:2025 - 검증되지 않은 외부 호출

### 설명:
검증되지 않은 외부 호출은 컨트랙트가 다른 컨트랙트나 주소에 외부 호출을 할 때 해당 호출의 결과를 적절히 확인하지 않는 보안 결함을 말합니다. 이더리움에서 컨트랙트가 다른 컨트랙트를 호출할 때, 호출된 컨트랙트는 예외를 발생시키지 않고 조용히 실패할 수 있습니다. 호출하는 컨트랙트가 반환 값을 확인하지 않으면, 실제로 실패했음에도 호출이 성공했다고 잘못 가정할 수 있습니다. 이로 인해 컨트랙트 상태의 불일치와 공격자가 악용할 수 있는 취약점이 발생할 수 있습니다.

### 예시 (취약한 컨트랙트):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        callee.delegatecall(_data);
    }
}
```
### 영향:
- 검증되지 않은 외부 호출은 거래 실패를 초래하여 의도된 작업이 성공적으로 완료되지 않을 수 있습니다. 이로 인해 컨트랙트가 전송이 성공했다고 잘못 가정하여 자금 손실이 발생할 수 있습니다. 또한, 잘못된 컨트랙트 상태를 생성하여 컨트랙트가 추가 악용 및 로직 불일치에 취약해질 수 있습니다.

### 해결 방안:
- 가능하면 send() 대신 transfer()를 사용합니다. transfer()는 외부 호출이 실패하면 거래를 되돌리기 때문입니다.
- send() 또는 call() 함수의 반환 값을 항상 확인하여 false를 반환할 경우 적절히 처리합니다.

### 예시 (수정된 버전):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_CheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // delegatecall이 성공하는지 확인
        (bool success, ) = callee.delegatecall(_data);
        require(success, "Delegatecall failed");  // 실패를 처리하기 위해 반환 값 확인
    }
}
```

### 주의사항

위의 두 컨트랙트는 검증되지 않은 반환 값 외에도 약점을 포함하고 있습니다.

- 인증이 callee에 위임됩니다. 호출된 컨트랙트의 코드는 `msg.sender`를 제한할 수도 있고 아닐 수도 있습니다(예: `owner`와 비교). 일반적으로 `forward` 함수는 어떤 형태의 인증을 수행해야 합니다.
- `callee`는 사용자가 제공한 주소입니다. 이는 이 컨트랙트의 컨텍스트에서 임의의 코드가 실행될 수 있음을 의미하며, 예를 들어 `owner`를 수정할 수 있습니다. 이는 `forward`가 인증을 수행하지 않기 때문에 특히 문제가 됩니다.
- `callee` 주소가 컨트랙트인지 확인되지 않습니다. `callee`가 코드가 없는 주소라면, `delegatecall`이 성공하므로 이 사실이 감지되지 않습니다. 일반적으로 `forward` 함수는 호출된 컨트랙트의 코드 크기가 0보다 큰지 확인하는 등의 기본 검사를 수행해야 합니다.

### 검증되지 않은 외부 호출 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Punk Protocol 해킹](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : 상세 [해킹 분석](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)
