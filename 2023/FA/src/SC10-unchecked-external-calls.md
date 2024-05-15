# Unchecked External Calls

### Description
In Ethereum, when a contract calls another contract, the called contract can fail silently without throwing an exception. If the calling contract doesn't check the outcome of the call, it might assume that the call was successful, even if it wasn't.

### Impact
Unchecked external calls can lead to failed transactions, lost funds, or incorrect contract state.

### Steps to Fix
1. Always check the return value of `call`, `delegatecall`, and `callcode`.
2. Use Solidity's `transfer` or `send` functions instead of `call.value()()`, as they automatically reverts on failure.

### Example
A contract uses the `call` function to send Ether to an address. If the call fails (for example, if the recipient is a contract without a payable fallback function), the sending contract might incorrectly assume the transfer was successful.
