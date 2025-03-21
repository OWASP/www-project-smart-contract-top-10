## SC09:2025 - 不安全的随机性

### 漏洞描述:
随机数生成器在赌博、游戏赢家选择和随机种子生成等应用中至关重要。然而，在以太坊上生成随机数具有挑战性，因为以太坊的执行环境是确定性的。Solidity 无法生成真正的随机数，而是依赖伪随机因素。此外，Solidity 中的复杂计算会消耗大量 gas，因此随机数的生成需要谨慎设计。

*Solidity 中不安全的随机数生成机制：开发者常使用与区块相关的方法生成随机数，例如：*
  - block.timestamp: 当前区块的时间戳。
  - blockhash(uint blockNumber): 指定区块的哈希值（仅适用于最近 256 个区块）。
  - block.difficulty: 当前区块的难度值。
  - block.number: 当前区块编号。
  - block.coinbase: 当前区块矿工的地址。
    
这些方法容易受到矿工操控，从而影响智能合约的逻辑。例如，矿工可以选择延迟出块或选择有利于自己的区块哈希值，以控制随机数的输出，导致合约受到攻击。

### 举例 (包含漏洞的合约):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_InsecureRandomness {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // Using insecure mechanisms for random number generation
            ) 
        );

        if (_guess == answer) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```
### 漏洞影响:
- 不安全的随机性可能被攻击者利用，导致在游戏、彩票或任何其他依赖随机数生成的智能合约中获得不公平的优势。通过预测或操控本应随机的结果，攻击者可以使结果对自己有利。这可能导致：

  - 不公平的获胜结果，破坏游戏或彩票的公正性。
  - 其他参与者的资金损失，攻击者可能通过操控结果窃取奖励或资金。
  - 对智能合约的信任缺失，导致用户对合约的完整性和公平性产生质疑。

### 修复方法:
- 使用预言机（Oraclize）作为外部随机性来源。— 应谨慎信任预言机服务，建议使用多个预言机进行交叉验证，以增强安全性。
- 使用承诺方案（Commitment Schemes）— 这是一种加密原语，采用**提交-揭示（commit-reveal）**机制，有效避免单方操控结果，广泛应用于投币、零知识证明和安全计算等场景。例如：RANDAO。
- Chainlink VRF（可验证随机函数）— Chainlink VRF 是一种可证明公平且可验证的随机数生成器（RNG），允许智能合约获取随机值而不妥协安全性或可用性。
- Signidice 算法 — 适用于涉及两方的伪随机数生成（PRNG），通过加密签名确保随机性。
- 比特币区块哈希（Bitcoin Block Hashes）— 可通过像 BTCRelay 这样的预言机，作为以太坊与比特币之间的桥梁，以太坊上的合约可以请求比特币区块链的未来区块哈希作为熵源。但需注意，这种方法无法避免矿工激励问题，应谨慎实施。

### 举例 (已修复版本):

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

### 遭受不安全随机性攻击的案例:
1. [Roast Football Hack](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST Hack](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)