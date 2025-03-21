## SC08:2025 - 整数溢出与下溢

### 漏洞描述:
以太坊虚拟机（EVM）为整数定义了固定大小的数据类型，这意味着整数变量所能表示的数值范围是有限的。例如，uint8（8 位无符号整数，即非负数）只能存储 0 到 255 之间的整数。如果尝试存储大于 255 的值，就会发生溢出（Overflow）。同样，如果从 0 减去 1，结果将变为 255，这称为下溢（Underflow）。

当算术运算超出整数类型的最大或最小值时，就会发生溢出或下溢。对于有符号整数（int），情况稍有不同。例如，如果对 int8 类型的 -128 进行 -1 运算，结果将变为 127，因为有符号整数类型在达到最小值时会循环回最大值。

类似的现象可以在现实世界中找到，例如：

周期性数学函数（如将输入值增加 2π 时，sin 函数的输出保持不变）。

汽车里程表（当数值超过最大值 999999 时，里程表会重置为 000000）。

***重要说明 :
在 Solidity `0.8.0` 及以上版本中，编译器会自动检查算术运算中的溢出和下溢，并在发生时回滚交易。 Solidity `0.8.0` 引入了 `unchecked` 关键字，允许开发者在不进行溢出检查的情况下执行算术运算。这在某些情况下（如 gas 优化或特定需求）可能有用，使得算术行为与旧版本 Solidity 相似。***

### 举例 (包含漏洞的合约):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.17;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() public {
        balance = 255; // uint8 的最大值
    }

    // Increments the balance by a given value
    function increment(uint8 value) public {
        balance += value; // 易受溢出攻击
    }

    // Decrements the balance by a given value
    function decrement(uint8 value) public {
        balance -= value; // 易受下溢攻击
    }
}

```
### 漏洞影响:
- 攻击者可以利用这些漏洞人为增加账户余额或代币数量，从而提取超过其合法拥有的资金。
- 攻击者可能篡改合约逻辑，导致未经授权的操作，如窃取资产或铸造过量代币。

### 修复方法:
- 使用 Solidity 0.8.0 及以上版本，确保编译器自动执行溢出和下溢检查。
- 使用安全数学库（Safe Math Libraries）：
  - 以太坊社区（Ethereum Community）提供了一些经过审计的安全库，如 OpenZeppelin 的 SafeMath。
  - SafeMath 提供 `add()`、`sub()`、`mul()` 等函数，在发生溢出或下溢时自动回滚交易，从而有效防止漏洞。

### 举例 (已修复版本):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() {
        balance = 255; // uint8 的最大值
    }

    // Solidity 0.8.x 自动检查溢出
    function increment(uint8 value) public {
        balance += value; // Solidity 0.8.x automatically checks for overflow
    }

    // 按指定数值减少余额
    function decrement(uint8 value) public {
        require(balance >= value, "Underflow detected");
        balance -= value;
    }
}
```

### 整数溢出与下溢攻击的案例:
1. [PoWH Coin Ponzi Scheme](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)