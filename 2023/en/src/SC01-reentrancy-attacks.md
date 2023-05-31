# Reentrancy Attacks

### Description
A reentrancy attack happens when a function is externally invoked during its execution, allowing it to be run multiple times in a single transaction. This typically occurs when a contract calls another contract before it resolves its state.

### Impact
A successful reentrancy attack can lead to fund drains, unauthorized function calls, or state changes that disrupt the normal operations of the contract.

### Steps to Fix
1. Make sure you follow the Checks-Effects-Interactions (CEI) pattern: check conditions, then make changes, then interact with other contracts.
2. Use a reentrancy guard or mutual exclusion lock (mutex) to block recursive calls from external contracts during a function's execution.
3. Regularly update to the latest version of Solidity, which includes inherent protection against reentrancy attacks.

### Example
The infamous DAO hack was a reentrancy attack. An attacker exploited a reentrancy vulnerability to drain around 3.6 million Ether from the contract.
