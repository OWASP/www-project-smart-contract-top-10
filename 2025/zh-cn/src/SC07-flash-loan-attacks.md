## SC07:2025 - 闪电贷攻击 

### 漏洞描述: 
闪电贷攻击利用了无需抵押即可在单笔交易中借入大量资金的机制。这种攻击利用了区块链交易的原子性，即所有操作要么成功，要么失败。攻击者通过结合闪电贷与其他漏洞（如预言机操纵、重入攻击或逻辑错误），可以操控合约行为并窃取资金。

#### 闪电贷攻击示例:
1. **预言机操纵:**   利用借入的资金扭曲价格预言机数据，触发超额清算。 
2. **流动性池耗尽:** 通过闪电贷移除流动性或利用设计不良的 AMM 机制。 
3. **套利攻击:** 操纵流动性，利用跨平台价格差异进行套利。

### 漏洞影响:
- **资金损失:** 攻击者可以耗尽协议储备，或通过操控抵押贷款窃取资产。 
- **市场扰动:**  短期的价格操控或流动性枯竭可能影响用户和平台。
- **生态系统损害:** 用户失去对协议的信任，导致用户减少和财务损失。 

### 修复方法:
- **避免在关键逻辑中依赖闪电贷:** 限制敏感功能，仅在经过验证和可预测的条件下运行。 
- **强化预言机设计:** 使用时间加权平均价格（TWAP）或抗操控的去中心化预言机。
- **全面测试:** 在测试中模拟闪电贷场景和边界情况。
- **访问控制:** 限制对关键功能的访问，防止未经授权或恶意交易。

### 遭受闪电贷攻击的案例:
1. [UwUlend Hack](https://blog.solidityscan.com/uwulend-hack-analysis-77eb9181a717): A Comprehensive [Hack Analysis](https://blog.solidityscan.com/uwulend-hack-analysis-77eb9181a717)
2. [Doughfina Hack](https://blog.solidityscan.com/doughfina-hack-analysis-685ed56adb19): A Comprehensive [Hack Analysis](https://blog.solidityscan.com/doughfina-hack-analysis-685ed56adb19)