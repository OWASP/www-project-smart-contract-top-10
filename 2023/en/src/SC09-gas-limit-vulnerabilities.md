# Gas Limit Vulnerabilities

### Description
If a function requires more gas than the block gas limit to complete its execution, it will inevitably fail. These vulnerabilities typically occur in loops that iterate over dynamic data structures.

### Impact
Functions vulnerable to gas limits can become uncallable, locking funds or freezing contract state.

### Steps to Fix
1. Avoid loops that iterate over dynamic data structures. If possible, use mappings and keep track of keys separately.
2. Implement gas-efficient code and test functions with large inputs to ensure they won't exceed the block gas limit.
3. Break down complex computations into multiple transactions if needed.

### Example
A token smart contract that implements a function to transfer all tokens of an array of addresses could exceed the block gas limit if the array is too large.
