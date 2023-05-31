# Logic Errors

### Description
Logic errors occur when the contract's code doesn't correctly implement the intended logic. This might be due to misunderstanding, error, or omission on the developer's part.

### Impact
Logic errors can cause the contract to behave unexpectedly or even become entirely unusable. They can lead to the loss of funds, incorrect distribution of tokens, or other adverse outcomes.

### Steps to Fix
1. Use automated testing frameworks to write extensive unit tests covering all possible edge cases.
2. Conduct thorough code reviews and audits.
3. Document the intended behavior of each function and module, then compare it to the actual implementation.

### Example
The parity multi-sig wallet had a logic error that allowed a user to take ownership of a library contract and self-destruct it, which indirectly caused the freezing of funds in all dependent contracts.
