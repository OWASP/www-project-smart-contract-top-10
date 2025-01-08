## SC06:2025  Unchecked External Calls

### Description:
Unchecked external calls refer to a security flaw where a contract makes an external call to another contract or address without properly checking the outcome of that call. In Ethereum, when a contract calls another contract, the called contract can fail silently without throwing an exception. If the calling contract doesn’t check the return value, it might incorrectly assume the call was successful, even if it wasn't. This can lead to inconsistencies in the contract state and vulnerabilities that attackers can exploit.

### Example (Vulnerable contract):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.24;

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    function forward(address callee, bytes _data) public {
        require(callee.delegatecall(_data));
    }
}
```
### Impact:
- Unchecked external calls can result in failed transactions, causing the intended operations to not be completed successfully. This can lead to the loss of funds, as the contract may proceed under the false assumption that the transfer was successful. Additionally, it can create an incorrect contract state, making the contract vulnerable to further exploits and inconsistencies in its logic.

### Remediation:
- Whenever possible, use transfer() instead of send(), as transfer() reverts the transaction if the external call fails.
- Always check the return value of send() or call() functions to ensure proper handling if they return false.

### Example (Fixed version):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // Ensure that delegatecall succeeds
        (bool success, ) = callee.delegatecall(_data);
        require(success, "Delegatecall failed");  // Check the return value to handle failure
    }
}
```

### Examples of Smart Contracts That Fell Victim to Unchecked External Call Attacks:
1. [Punk Protocol Hack](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)