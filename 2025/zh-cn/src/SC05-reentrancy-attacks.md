## SC05:2025 - 重入攻击

### 漏洞描述:
重入攻击（reentrancy attack）利用了智能合约中的一个漏洞，即当函数在更新自身状态之前向另一个合约发起外部调用时，可能导致外部合约（可能是恶意的）重新进入原始函数，并利用相同的状态重复某些操作，如提款。通过这种攻击，攻击者有可能耗尽合约中的所有资金。

### 举例 (包含漏洞的合约): 
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // 漏洞：在更新用户余额之前发送以太币，允许重入攻击。
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // 发送以太后更新余额
        balances[msg.sender] = 0;
    }
}
```
### 漏洞影响:
- 最直接且最严重的后果是资金被耗尽。攻击者利用漏洞提取超出其权限的资金，可能会完全清空合约余额。
- 攻击者可以触发未授权的函数调用，导致合约或相关系统执行意外操作。

### 修复方法:
- 始终确保在调用外部合约之前进行状态更改，即先更新内部余额或执行内部逻辑，再发起外部调用。
- 使用防止重入攻击的函数修饰符，如OpenZeppelin提供的重入防护（Re-entrancy Guard）。

### 举例 (已修复版本):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // 修复方法：先更新用户余额，然后再发送以太币。
        balances[msg.sender] = 0;

        // 更新后再发送以太币
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### 遭受重入攻击的案例:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)