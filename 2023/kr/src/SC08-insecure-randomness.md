## 취약점: 안전하지 않은 난수 생성

### 설명:
난수 생성기는 도박, 게임 승자 선정, 랜덤 시드 생성과 같은 애플리케이션에 필수적입니다. 이더리움에서 난수를 생성하는 것은 결정론적 특성 때문에 어렵습니다. Solidity는 진정한 난수를 생성할 수 없기 때문에 의사 난수 요소에 의존합니다. 또한 Solidity에서 복잡한 계산은 가스 측면에서 비용이 많이 듭니다.

*Solidity에서 안전하지 않은 메커니즘으로 난수 생성: 개발자들은 종종 블록 관련 방법을 사용하여 난수를 생성합니다:*
  - block.timestamp: 현재 블록 타임스탬프
  - blockhash(uint blockNumber): 주어진 블록의 해시 (최근 256개 블록에만 해당)
  - block.difficulty: 현재 블록 난이도
  - block.number: 현재 블록 번호
  - block.coinbase: 현재 블록 채굴자의 주소

이러한 방법들은 채굴자가 조작할 수 있어 컨트랙트의 로직에 영향을 줄 수 있으므로 안전하지 않습니다.

### 예시:
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract InsecureRandomNumber {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // 난수 생성에 안전하지 않은 메커니즘 사용
            ) 
        );

        if (_guess == answer) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```
### 영향:
- 안전하지 않은 난수성은 공격자가 게임, 복권 및 난수 생성에 의존하는 기타 컨트랙트에서 부당한 이점을 얻기 위해 악용될 수 있습니다. 추정상 무작위인 결과를 예측하거나 조작함으로써 공격자는 자신에게 유리하게 결과에 영향을 줄 수 있습니다. 이는 불공정한 승리, 다른 참가자의 재정적 손실, 스마트 컨트랙트의 무결성과 공정성에 대한 전반적인 신뢰 부족으로 이어질 수 있습니다.

### 해결 방안:
- 오라클(Oraclize)을 외부 난수 소스로 사용합니다. 오라클을 신뢰할 때 주의해야 합니다. 여러 오라클을 사용할 수도 있습니다.
- 커밋먼트 방식 사용 — 커밋-공개 방식을 사용하는 암호화 기본 요소를 따를 수 있습니다. 동전 던지기, 영지식 증명, 보안 계산에서도 광범위하게 적용됩니다. 예: RANDAO.
- Chainlink VRF — 보안이나 사용성을 손상시키지 않고 스마트 컨트랙트가 난수 값에 접근할 수 있게 하는 입증 가능하게 공정하고 검증 가능한 난수 생성기(RNG)입니다.
- Signidice 알고리즘 — 암호화 서명을 사용하는 두 당사자가 관련된 애플리케이션의 PRNG에 적합합니다.
- 비트코인 블록 해시 — 이더리움과 비트코인 사이의 브릿지 역할을 하는 BTCRelay와 같은 오라클을 사용할 수 있습니다. 이더리움의 컨트랙트는 비트코인 블록체인에서 미래 블록 해시를 엔트로피 소스로 요청할 수 있습니다. 이 접근 방식은 채굴자 인센티브 문제에 대해 안전하지 않으며 주의해서 구현해야 합니다.

### 안전하지 않은 난수성 공격 피해 사례:
1. [Roast Football 해킹](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST 해킹](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : 상세한 [해킹 분석](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)
