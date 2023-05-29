# Insecure Randomness

### Description
Generating randomness in Ethereum is challenging because every node must come to the same conclusion on the state of the blockchain. Hence, naive approaches to generate randomness can be manipulated by miners or observant attackers.

### Impact
Insecure randomness can be exploited by attackers to gain an unfair advantage in games, lotteries, or any other contracts that rely on random number generation.

### Steps to Fix
1. Use commit-reveal schemes, where users submit hashed values and reveal them later, to generate randomness.
2. Use external oracle services that provide random numbers.

### Example
A lottery smart contract using `block.timestamp` for generating a random number can be manipulated by a miner, making the lottery unfair.

