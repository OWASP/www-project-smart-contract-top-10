## SC06:2025  Unchecked External Calls

### Description:
Unchecked external calls refer to a security flaw where a contract makes an external call to another contract or address without properly checking the outcome of that call. In Ethereum, when a contract calls another contract, the called contract can fail silently without throwing an exception. If the calling contract doesn’t check the return value, it might incorrectly assume the call was successful, even if it wasn't. This can lead to inconsistencies in the contract state and vulnerabilities that attackers can exploit.

### Example (Vulnerable contract):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        callee.delegatecall(_data);
    }
}
```
### Impact:
- Unchecked external calls can result in failed transactions, causing the intended operations to not be completed successfully. This can lead to the loss of funds, as the contract may proceed under the false assumption that the transfer was successful. Additionally, it can create an incorrect contract state, making the contract vulnerable to further exploits and inconsistencies in its logic.

### Remediation:
- Whenever possible, use transfer() instead of send(), as transfer() reverts the transaction if the external call fails.
- Always check the return value of send() or call() functions to ensure proper handling if they return false.

### Example (Fixed version):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_CheckedExternalCall {
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

### Caveats

The two contracts above contain weaknesses beyond an unchecked return value.

- Authentication is delegated to the callee. The code of the called contract may, or may not, restrict the `msg.sender`, e.g. by comparing it to `owner`. Normally, the function `forward` should perform some form of authentication.
- `callee` is an address provided by the user. This means that arbitrary code can be executed in the context of this contract, modifying e.g.\ `owner`. This is particularly problematic, as `forward` does not perform authentication.
- The address `callee` is not checked for being a contract. If `callee` is an address without code, this will go unnoticed, as `delegatecall` succeeds. Normally, the function `forward` should do basic checks, like verifying that the code size of the called contract is larger than zero.

### Examples of Smart Contracts That Fell Victim to Unchecked External Call Attacks:
1. [Punk Protocol Hack](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)