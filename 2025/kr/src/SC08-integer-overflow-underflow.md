## SC08:2025 - 정수 오버플로우 및 언더플로우

### 설명:
이더리움 가상 머신(EVM)은 정수에 대해 고정 크기 데이터 타입을 정의합니다. 이는 정수 변수가 나타낼 수 있는 숫자의 범위가 유한하다는 것을 의미합니다. 예를 들어, "uint8"(8비트 부호 없는 정수, 즉 음이 아닌 정수)은 0에서 255 사이의 정수만 저장할 수 있습니다. 255보다 큰 값을 "uint8"에 저장하려고 하면 오버플로우가 발생합니다. 마찬가지로, "0"에서 "1"을 빼면 255가 됩니다. 이를 언더플로우라고 합니다. 산술 연산이 타입의 최대 또는 최소 크기를 초과하거나 미달할 때 오버플로우 또는 언더플로우가 발생합니다. 부호 있는 정수의 경우 결과가 약간 다릅니다. 값이 -128인 int8에서 "1"을 빼면 127이 됩니다. 이는 음수 값을 나타낼 수 있는 부호 있는 int 타입이 가장 높은 음수 값에 도달하면 다시 시작하기 때문입니다. 이 동작의 두 가지 간단한 예는 주기적인 수학 함수(sin의 인수에 2를 더해도 값이 그대로 유지됨)와 자동차의 주행 거리계(거리를 추적하는 장치로, 최대 숫자인 999999를 초과하면 000000으로 리셋됨)입니다.

***중요 참고사항:-
Solidity `0.8.0` 이상에서는 컴파일러가 산술 연산에서 오버플로우와 언더플로우를 자동으로 확인하고, 오버플로우나 언더플로우가 발생하면 트랜잭션을 되돌립니다.
Solidity `0.8.0`은 또한 `unchecked` 키워드를 도입하여 개발자가 이러한 자동 검사 없이 산술 연산을 수행할 수 있게 하고, 되돌리지 않고 오버플로우를 명시적으로 허용합니다. 이는 오버플로우가 우려되지 않는 경우나 이전 버전의 Solidity에서 산술이 작동하던 방식과 유사하게 래핑 동작이 필요한 경우 가스 사용량을 최적화하는 데 특히 유용할 수 있습니다.***

### 예시 (취약한 컨트랙트):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.17;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() public {
        balance = 255; // uint8의 최대값
    }

    // 주어진 값만큼 잔액을 증가시킴
    function increment(uint8 value) public {
        balance += value; // 오버플로우에 취약
    }

    // 주어진 값만큼 잔액을 감소시킴
    function decrement(uint8 value) public {
        balance -= value; // 언더플로우에 취약
    }
}

```
### 영향:
- 공격자가 이러한 취약점을 악용하여 계정 잔액이나 토큰 수량을 인위적으로 증가시켜, 정당하게 소유한 것보다 더 많은 자금을 인출할 수 있습니다.
- 공격자가 컨트랙트 로직의 의도된 흐름을 변경하여 자산 도난이나 과도한 토큰 발행과 같은 무단 작업을 수행할 수 있습니다.

### 해결 방안:
- 가장 간단한 접근 방식은 Solidity 컴파일러 버전 0.8.0 이상을 사용하는 것입니다. 이 버전은 오버플로우와 언더플로우 검사를 자동으로 처리합니다.
- 최신 Safe Math 라이브러리 사용: 이더리움 커뮤니티에서 OpenZeppelin은 안전한 라이브러리를 만들고 감사하는 데 훌륭한 일을 해왔습니다. 특히 SafeMath 라이브러리를 사용하여 언더플로우/오버플로우 취약점을 방지할 수 있습니다. 이 라이브러리는 add(), sub(), mul() 등의 함수를 제공하여 기본 산술 연산을 수행하고 오버플로우나 언더플로우가 발생하면 자동으로 되돌립니다.

### 예시 (수정된 버전):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() {
        balance = 255; // uint8의 최대값
    }

    // 주어진 값만큼 잔액을 증가시킴
    function increment(uint8 value) public {
        balance += value; // Solidity 0.8.x는 자동으로 오버플로우를 확인함
    }

    // 주어진 값만큼 잔액을 감소시킴
    function decrement(uint8 value) public {
        require(balance >= value, "Underflow detected");
        balance -= value;
    }
}
```

### 정수 오버플로우 및 언더플로우 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [PoWH Coin 폰지 스킴](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : 상세 [해킹 분석](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : 상세 [해킹 분석](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)
