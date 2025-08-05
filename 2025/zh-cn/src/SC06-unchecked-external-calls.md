## SC06:2025  未经检查的外部调用

### 漏洞描述:
未经检查的外部调用是一种安全漏洞，指的是合约在调用另一个合约或地址时，没有正确验证调用结果。在以太坊中，当一个合约调用另一个合约时，被调用的合约可能会默默失败，而不会抛出异常。如果调用合约没有检查返回值，就可能错误地假设调用成功，尽管实际情况是失败的。这种情况可能导致合约状态不一致，并为攻击者提供利用漏洞的机会。

### 举例 (包含漏洞的合约):
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
### 漏洞影响:
- 未检查的外部调用可能导致交易失败，使预期的操作无法顺利完成。这可能会导致资金损失，因为合约可能会错误地假设转账成功并继续执行。此外，这还可能导致合约状态不正确，从而进一步暴露于攻击和逻辑不一致的风险。

### 修复方法:
- 在可能的情况下，优先使用 `transfer()` 而非 `send()`，因为 `transfer()` 在外部调用失败时会回滚交易。
- 始终检查 `send()` 或 `call()` 的返回值，并在返回 `false` 时进行适当处理。

### 举例 (已修复版本):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_CheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // 确保 delegatecall 调用成功
        (bool success, ) = callee.delegatecall(_data);
        require(success, "Delegatecall failed");  // Check the return value to handle failure
    }
}
```

### 遭受未受检查外部调用攻击的案例:
1. [Punk Protocol Hack](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)