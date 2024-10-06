## آسیب‌پذیری: حمله Denial Of Service

### توضیحات:
حمله Denial of Service (DoS) در سالیدیتی شامل سوءاستفاده از آسیب‌پذیری‌ها برای تخلیه منابعی مانند GAS، چرخه‌های CPU یا ذخیره‌سازی است که منجر به غیرقابل استفاده شدن یک قرارداد هوشمند می‌شود. انواع رایج این حملات شامل موارد زیر است:

حملات تخلیه GAS: در این نوع حملات، بازیگران مخرب تراکنش‌هایی ایجاد می‌کنند که نیاز به GAS بیش از حد دارند، که منجر به تخلیه منابع می‌شود.

حملات بازگشتی: این نوع حملات از توالی تماس‌های قرارداد سوءاستفاده می‌کند تا به وجوه غیرمجاز دسترسی پیدا کند.

حملات محدودیت GAS بلوک: در این نوع حملات، منابع GAS بلوک مصرف می‌شود و این موضوع مانع از انجام تراکنش‌های مشروع می‌شود.

### Example :
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableKingOfEther {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");  // if the current king's fallback function reverts, it will prevent others from becoming the new king, causing a Denial of Service.

        balance = msg.value;
        king = msg.sender;
    }
}
```
### Impact:
- A successful DoS attack can render the smart contract unresponsive, preventing users from interacting with it as intended. This can disrupt critical operations and services relying on the contract.
-  DoS attacks can lead to financial losses, especially in decentralized applications (dApps) where smart contracts manage funds or assets.
- A DoS attack can tarnish the reputation of the smart contract and its associated platform. Users may lose trust in the platform's security and reliability, leading to a loss of users and business opportunities.
  
### Remediation:
- Ensure smart contracts can handle consistent failures, such as asynchronous processing of potentially failing external calls, to maintain contract integrity and prevent unexpected behavior.
- Be cautious when using `call` for external calls, loops, and traversals to avoid excessive gas consumption, which could lead to failed transactions or unexpected costs.
- Avoid over-authorizing a single role in contract permissions. Instead, divide permissions reasonably and use multi-signature wallet management for roles with critical permissions to prevent permission loss due to private key compromise.
