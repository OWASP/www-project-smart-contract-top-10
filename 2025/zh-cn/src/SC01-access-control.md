## SC01:2025 - 访问控制漏洞 

### 漏洞描述:
访问控制漏洞是一种安全缺陷，允许未经授权的用户访问或修改合约的数据或函数。这些漏洞通常源于合约代码未能根据用户权限级别实施足够的访问限制。在智能合约中，访问控制可能涉及治理和关键逻辑，例如铸造代币、投票治理提案、提取资金、暂停和升级合约，以及更改所有权等功能。

### 举例 (包含漏洞的合约):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_AccessControl {
    mapping(address => uint256) public balances;

    // 销毁函数没有进行权限控制，任何人都可以调用
    function burn(address account, uint256 amount) public {
        _burn(account, amount);
    }
}
```
### 漏洞影响:
- 攻击者可能通过访问合约内的关键函数和数据，获取未授权的权限，从而破坏合约的完整性和安全性。
- 这些漏洞可能导致合约控制的资金或资产被盗，进而对用户和利益相关者造成严重的经济损失。

### 修复方法:
- 确保初始化函数只能由授权实体调用一次，以防止未经授权的访问或修改。
- 在合约中使用已建立的访问控制模式，如Ownable或RBAC（基于角色的访问控制），来管理权限，确保只有授权用户才能访问特定的函数。这可以通过向敏感函数添加适当的访问控制修饰符来实现，例如 `onlyOwner` 或自定义角色。

### 举例 (已修复版本):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// 从OpenZeppelin导入Ownable合约以管理所有权
import "@openzeppelin/contracts/access/Ownable.sol";

contract Solidity_AccessControl is Ownable {
    mapping(address => uint256) public balances;

    // 具有合理访问控制的销毁函数，只能由所有者访问
    function burn(address account, uint256 amount) public onlyOwner {
        _burn(account, amount);
    }
}
```

### 遭受不当访问控制攻击的案例：
1. [HospoWise Hack](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT Hack](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)