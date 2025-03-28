# SC04:2025 - 缺乏输入验证 

## 漏洞描述:
输入验证确保智能合约只处理有效且符合预期的数据。未能验证传入的输入会使合约暴露于安全风险，如逻辑操控、未授权访问和意外行为。例如，如果合约假设用户输入始终有效而不进行验证，攻击者可以利用这一信任漏洞引入恶意数据。这种缺乏输入验证的缺陷会严重损害智能合约的安全性和可靠性。

## 举例 (包含漏洞的合约):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LackOfInputValidation {
    mapping(address => uint256) public balances;

    function setBalance(address user, uint256 amount) public {
        // 该函数允许任何人在没有验证的情况下为任意用户设置任意余额
        balances[user] = amount;
    }
}
```
### 漏洞影响:
- 攻击者可以操纵输入以耗尽资金、窃取代币或造成其他财务损失。 
- 不正确的输入可能会破坏状态变量，导致合约行为不可靠和不安全。
- 攻击者可能利用合约执行未授权的交易或操作，操控合约功能，进而影响用户和整个系统的安全性。

### 修复方法:
- 确保输入数据符合预期的类型。
- 验证输入是否在可接受的边界范围内。
- 确保只有授权实体能够调用特定的函数。
- 验证输入的结构是否正确，如地址格式或字符串长度。
- 当输入验证失败时，立即停止执行，并提供清晰的错误消息。

### 举例 (已修复版本):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LackOfInputValidation {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not authorized");
        _;
    }

    function setBalance(address user, uint256 amount) public onlyOwner {
        require(user != address(0), "Invalid address");
        balances[user] = amount;
    }
}
```
### 由于缺乏输入验证而成为攻击受害者的示例:
1. [Convergence Finance](https://etherscan.io/address/0x2b083beaaC310CC5E190B1d2507038CcB03E7606#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/convergence-finance-hack-analysis-12e6acd9ea08)
2. [Socket Gateway](https://etherscan.io/address/0x3a23F943181408EAC424116Af7b7790c94Cb97a5#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/socket-gateway-hack-analysis-b0e9567f7d3e)