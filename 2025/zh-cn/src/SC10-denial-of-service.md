## SC10:2025 - 拒绝服务（DoS）攻击

### 漏洞描述:
在 Solidity 中，拒绝服务（DoS）攻击通过利用漏洞耗尽资源，如 gas、CPU 周期或存储，导致智能合约无法正常使用。常见的攻击类型包括：

- Gas 耗尽攻击：恶意行为者创建需要过多 gas 的交易，导致合约无法执行。
- 重入攻击（Reentrancy）：攻击者通过操控合约调用顺序，访问未经授权的资金。
- 区块 gas 限制攻击：通过消耗区块中的 gas，攻击者阻碍合法交易的执行。

### 举例 (包含漏洞的合约):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        //如果当前的国王合约包含恶意的回退函数，并且该函数会触发回滚，那么新国王合约将无法成功继承王位，从而导致**拒绝服务（DoS）**攻击。换句话说，恶意回退函数会阻止新国王合约的执行，使得智能合约无法正常运行。
        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```
### 漏洞影响:
- 成功的 DoS 攻击可能导致智能合约失去响应，阻止用户按预期与合约交互。这会干扰依赖该合约的关键操作和服务。
-  DoS 攻击可能造成经济损失，尤其是在 **去中心化应用（dApp）** 中，智能合约用于管理资金或资产时。
- DoS 攻击还可能损害智能合约及其平台的声誉，用户可能失去对平台安全性和可靠性的信任，从而导致用户流失和商业机会的丧失。
  
### 修复方法:
- 确保智能合约能够处理一致的失败情况，例如异步处理可能失败的外部调用，以维护合约的完整性并防止意外行为。
- 小心使用 call 进行外部调用、循环和遍历，以避免过度消耗 gas，导致交易失败或产生额外成本。
- 避免在合约权限中对单一角色进行过度授权。合理划分权限，并对具有关键权限的角色使用多签名钱包管理，以防止因私钥泄露导致权限丧失。

### 举例 (已修复版本):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    //  使用更安全的资金转移方式，例如 transfer，它具有固定的 gas 补贴。
    // 这样可以避免使用 call，从而防止 '恶意回退函数(malicious fallback)' 引发的问题。
    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        address previousKing = king;
        uint256 previousBalance = balance;

        // 在转移 Ether 之前更新状态，以防止重入攻击问题。
        king = msg.sender;
        balance = msg.value;

        // 使用 transfer 代替 call，以确保交易不会因恶意回退函数而失败。
        payable(previousKing).transfer(previousBalance);
    }
}
```