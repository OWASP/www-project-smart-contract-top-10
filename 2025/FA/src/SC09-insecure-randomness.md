## SC09:2025 - Insecure Randomness

### Description:
Random number generators are essential for applications like gambling, game-winner selection, and random seed generation. On Ethereum, generating random numbers is challenging due to its deterministic nature. Since Solidity cannot produce true random numbers, it relies on pseudorandom factors. Additionally, complex calculations in Solidity are costly in terms of gas.

*Insecure Mechanisms Create Random Numbers in Solidity: Developers often use block-related methods to generate random numbers, such as:*
  - block.timestamp: Current block timestamp.
  - blockhash(uint blockNumber): Hash of a given block (only for the last 256 blocks).
  - block.difficulty: Current block difficulty.
  - block.number: Current block number.
  - block.coinbase: Address of the current block’s miner.
    
These methods are insecure because miners can manipulate them, affecting the contract’s logic.

### Example (Vulnerable contract):
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
### Impact:
- Insecure randomness can be exploited by attackers to gain an unfair advantage in games, lotteries, and any other contracts that rely on random number generation. By predicting or manipulating the supposedly random outcomes, attackers can influence the results in their favor. This can lead to unfair wins, financial losses for other participants, and a general lack of trust in the smart contract's integrity and fairness. 

### Remediation:
- Using oracles (Oraclize) as external sources of randomness. Care should be taken while trusting the Oracle. Multiple Oracles can also be used.
- Using Commitment Schemes — A cryptographic primitive that uses a commit-reveal approach can be followed. It also has wide applications in coin flipping, zero-knowledge proofs, and secure computation. E.g.: RANDAO.
- Chainlink VRF — It is a provably fair and verifiable random number generator (RNG) that enables smart contracts to access random values without compromising security or usability.
- The Signidice Algorithm — Suitable for PRNG in applications involving two parties using cryptographic signatures.
- Bitcoin Block Hashes — Oracles like BTCRelay can be used which act as a bridge between Ethereum and Bitcoin. Contracts on Ethereum can request future block hashes from the Bitcoin Blockchain as a source of entropy. It should be noted that this approach is not safe against the miner incentive problem and should be implemented with caution.

### Example (Fixed version):

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

### Examples of Smart Contracts That Fell Victim to Insecure Randomness Attacks:
1. [Roast Football Hack](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST Hack](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)