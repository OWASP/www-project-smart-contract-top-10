## SC09:2025 - 안전하지 않은 난수 생성

### 설명:
난수 생성기는 도박, 게임 승자 선정, 랜덤 시드 생성과 같은 애플리케이션에 필수적입니다. 이더리움에서는 결정론적 특성으로 인해 난수 생성이 어렵습니다. Solidity는 진정한 난수를 생성할 수 없으므로 의사 난수 요소에 의존합니다. 또한, Solidity에서의 복잡한 계산은 가스 측면에서 비용이 많이 듭니다.

*Solidity에서 난수를 생성하는 안전하지 않은 메커니즘: 개발자들은 종종 블록 관련 방법을 사용하여 난수를 생성합니다:*
  - block.timestamp: 현재 블록 타임스탬프.
  - blockhash(uint blockNumber): 주어진 블록의 해시 (최근 256개 블록에 대해서만).
  - block.difficulty: 현재 블록 난이도.
  - block.number: 현재 블록 번호.
  - block.coinbase: 현재 블록 채굴자의 주소.
    
이러한 방법들은 채굴자가 조작할 수 있어 컨트랙트의 로직에 영향을 미칠 수 있으므로 안전하지 않습니다.

### 예시 (취약한 컨트랙트):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_InsecureRandomness {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // 난수 생성을 위한 안전하지 않은 메커니즘 사용
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
- 안전하지 않은 난수는 공격자가 게임, 복권, 그리고 난수 생성에 의존하는 모든 컨트랙트에서 불공정한 이점을 얻는 데 악용될 수 있습니다. 가정된 랜덤 결과를 예측하거나 조작함으로써 공격자는 결과를 자신에게 유리하게 영향을 미칠 수 있습니다. 이는 불공정한 승리, 다른 참가자들의 재정적 손실, 그리고 스마트 컨트랙트의 무결성과 공정성에 대한 전반적인 신뢰 상실로 이어질 수 있습니다.

### 해결 방안:
- 오라클(Oraclize)을 외부 난수 소스로 사용 — 오라클을 신뢰할 때 주의해야 합니다. 여러 오라클을 사용할 수도 있습니다.
- 커밋먼트 스킴 사용 — 커밋-공개 접근 방식을 사용하는 암호화 기본 요소를 따를 수 있습니다. 이는 동전 던지기, 영지식 증명, 보안 계산에서도 광범위하게 적용됩니다. 예: RANDAO.
- Chainlink VRF — 스마트 컨트랙트가 보안이나 사용성을 손상시키지 않고 무작위 값에 접근할 수 있게 하는 증명 가능하고 검증 가능한 난수 생성기(RNG)입니다.
- Signidice 알고리즘 — 암호화 서명을 사용하는 두 당사자가 관련된 애플리케이션의 PRNG에 적합합니다.
- 비트코인 블록 해시 — 이더리움과 비트코인 사이의 다리 역할을 하는 BTCRelay와 같은 오라클을 사용할 수 있습니다. 이더리움의 컨트랙트는 엔트로피 소스로 비트코인 블록체인에서 미래 블록 해시를 요청할 수 있습니다. 이 접근 방식은 채굴자 인센티브 문제에 대해 안전하지 않으며 주의하여 구현해야 합니다.

### 예시 (수정된 버전):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract Solidity_InsecureRandomness is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;

    constructor(address _vrfCoordinator, address _linkToken, bytes32 _keyHash, uint256 _fee) 
        VRFConsumerBase(_vrfCoordinator, _linkToken) 
    {
        keyHash = _keyHash;
        fee = _fee;
    }

    function requestRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK");
        return requestRandomness(keyHash, fee);
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        randomResult = randomness;
    }

    function guess(uint256 _guess) public {
        require(randomResult > 0, "Random number not generated yet");
        if (_guess == randomResult) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```

### 안전하지 않은 난수 공격의 피해를 입은 스마트 컨트랙트 사례:
1. [Roast Football 해킹](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : 상세 [해킹 분석](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST 해킹](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : 상세 [해킹 분석](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)
