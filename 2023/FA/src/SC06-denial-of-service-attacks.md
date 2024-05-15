# Denial of Service (DoS) Attacks

### Description
Denial of Service (DoS) is an attack where an adversary manages to halt the normal operations of a smart contract, making it unavailable to users.

### Impact
A DoS attack can disrupt the functionality of the smart contract, preventing users from interacting with it, and in some cases, resulting in financial loss.

### Steps to Fix
1. Be careful when using the 'send' and 'transfer' functions as they can potentially cause a DoS attack. Use 'call' instead, and handle the potential 'false' return value.
2. Limit the number of iterations or actions that can be taken in a single transaction to avoid reaching the gas limit.
3. Implement pull payments for refunds or withdrawals, which separates the process of awarding and withdrawing funds into two separate transactions.

### Example
A smart contract auction could become a victim of a DoS attack if the highest bidder is a malicious contract that always throws an exception when the contract tries to refund the second-highest bidder.
