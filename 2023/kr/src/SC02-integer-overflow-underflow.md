## 취약점: 정수 오버플로우 및 언더플로우

### 설명:
이더리움 가상 머신(EVM)은 정수에 대해 고정 크기 데이터 타입을 정의합니다. 이는 정수 변수가 나타낼 수 있는 숫자의 범위가 유한하다는 것을 의미합니다. 예를 들어, "uint8"(8비트 부호 없는 정수, 즉 음수가 아닌 정수)은 0에서 255 사이의 정수만 저장할 수 있습니다. 255보다 큰 값을 "uint8"에 저장하려고 하면 오버플로우가 발생합니다. 마찬가지로 "0"에서 "1"을 빼면 255가 됩니다. 이를 언더플로우라고 합니다.

산술 연산이 타입의 최대 또는 최소 크기를 초과하거나 미달하면 오버플로우 또는 언더플로우가 발생합니다. 부호 있는 정수의 경우 결과가 약간 다릅니다. 값이 -128인 int8에서 "1"을 빼면 127이 됩니다. 이는 음수 값을 나타낼 수 있는 부호 있는 정수 타입이 가장 높은 음수 값에 도달하면 다시 시작되기 때문입니다.

이 동작의 두 가지 간단한 예로는 주기적 수학 함수(sin의 인수에 2를 더해도 값이 변하지 않음)와 자동차의 주행거리계(최대 숫자인 999999를 초과하면 000000으로 초기화됨)가 있습니다.

***중요 참고 사항:-
Solidity `0.8.0` 이상에서는 컴파일러가 산술 연산에서 오버플로우와 언더플로우 검사를 자동으로 처리하며, 오버플로우 또는 언더플로우가 발생하면 트랜잭션을 되돌립니다.
Solidity `0.8.0`은 또한 `unchecked` 키워드를 도입하여 개발자가 이러한 자동 검사 없이 산술 연산을 수행할 수 있게 하며, 오버플로우가 발생해도 되돌리지 않고 명시적으로 허용합니다. 이는 오버플로우가 문제가 되지 않거나 이전 버전의 Solidity에서 산술이 동작했던 것처럼 래핑(wrap-around) 동작이 필요한 경우 가스 사용을 최적화하는 데 특히 유용할 수 있습니다.***

### 예시:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.0;

contract TimeWrapVault {
    mapping(address => uint256) public accountBalances;
    mapping(address => uint256) public withdrawalUnlockTime;

    // 누구나 ETH를 예치하고 잠금 시간을 설정할 수 있음
    function depositFunds() external payable {
        accountBalances[msg.sender] += msg.value;
        withdrawalUnlockTime[msg.sender] = block.timestamp + 1 weeks;
    }

    // 오버플로우에 취약함, 사용자가 매우 큰 값을 전달하여 오버플로우를 유발할 수 있음
    function extendLockTime(uint256 _additionalSeconds) public {
        withdrawalUnlockTime[msg.sender] += _additionalSeconds;
    }

    // 현재 시간이 잠금 시간보다 크면 출금 허용
    function releaseFunds() public {
        require(accountBalances[msg.sender] > 0, "Insufficient funds");
        require(block.timestamp > withdrawalUnlockTime[msg.sender], "Lock time not expired");

        uint256 withdrawalAmount = accountBalances[msg.sender];
        accountBalances[msg.sender] = 0;

        (bool successfulWithdrawal,) = msg.sender.call{value: withdrawalAmount}("");
        require(successfulWithdrawal, "Failed to send Ether");
    }
}

```
### 영향:
- 공격자는 이러한 취약점을 악용하여 계정 잔액이나 토큰 수량을 인위적으로 증가시켜, 합법적으로 소유한 것보다 더 많은 자금을 인출할 수 있습니다.
- 공격자가 의도된 컨트랙트 로직의 흐름을 변경하여 자산 탈취나 과도한 토큰 발행과 같은 승인되지 않은 행위를 수행할 수 있습니다.

### 해결 방안:
- 가장 간단한 방법은 Solidity 컴파일러 버전 0.8.0 이상을 사용하는 것입니다. 이 버전은 오버플로우 및 언더플로우 검사를 자동으로 처리합니다.
- 최신 Safe Math 라이브러리 사용: 이더리움 커뮤니티에서 OpenZeppelin은 안전한 라이브러리를 만들고 감사하는 데 훌륭한 역할을 해왔습니다. 특히 SafeMath 라이브러리는 언더플로우/오버플로우 취약점을 방지하는 데 사용할 수 있습니다. add(), sub(), mul() 등과 같은 기본 산술 연산을 수행하고 오버플로우 또는 언더플로우가 발생하면 자동으로 되돌리는 함수를 제공합니다.

### 정수 오버플로우 및 언더플로우 공격 피해 사례:
1. [PoWH Coin 폰지 사기](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)
