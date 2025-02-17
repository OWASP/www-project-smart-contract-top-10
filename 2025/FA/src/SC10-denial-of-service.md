## SC10:2025 - Denial Of Service

### توضیحات:
حمله Denial of Service (DoS) در سالیدیتی شامل سوءاستفاده از آسیب‌پذیری‌ها برای تخلیه منابعی مانند GAS، چرخه‌های CPU یا ذخیره‌سازی است که منجر به غیرقابل استفاده شدن یک قرارداد هوشمند می‌شود. انواع رایج این حملات شامل موارد زیر است:

حملات تخلیه گس: در این نوع حملات، بازیگران مخرب تراکنش‌هایی ایجاد می‌کنند که نیاز به گس بیش از حد دارند، که منجر به تخلیه منابع می‌شود.

حملات بازگشتی: این نوع حملات از توالی تماس‌های قرارداد سوءاستفاده می‌کند تا به وجوه غیرمجاز دسترسی پیدا کند.

حملات محدودیت گس بلاک: در این نوع حملات، منابع گس بلاک مصرف می‌شود و این موضوع مانع از انجام تراکنش‌های مشروع می‌شود.

### مثال (قرارداد اصلاح شده):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        //If the current king has a malicious fallback function that reverts, it will prevent the new king from claiming the throne, causing a Denial of Service.
        (bool sent,) = king.call{value: balance}("");
        require(sent, "Failed to send Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```
### شدت آسیب‌پذیری:
- یک حمله DoS موفق می‌تواند قرارداد هوشمند را غیرقابل پاسخگویی کند و کاربران را از تعامل با آن طبق انتظار بازدارد. این موضوع می‌تواند باعث اختلال در عملیات و خدمات حیاتی که به قرارداد وابسته هستند، شود.
-  حملات DoS می‌توانند منجر به خسارات مالی شوند، به‌ویژه در برنامه‌های غیرمتمرکز (dApps) که قراردادهای هوشمند در آن‌ها مدیریت وجوه یا دارایی‌ها را بر عهده دارند.
- یک حمله DoS می‌تواند به اعتبار قرارداد هوشمند و پلتفرم مرتبط با آن آسیب بزند. کاربران ممکن است اعتماد خود را به امنیت و قابلیت اطمینان پلتفرم از دست بدهند و این موضوع منجر به از دست دادن کاربران و فرصت‌های تجاری شود.
  
### راهکارهای امنیتی:
- اطمینان حاصل کنید که قراردادهای هوشمند می‌توانند خرابی‌های مداوم، مانند پردازش ناهمزمان فراخوانی‌های خارجی که ممکن است شکست بخورند، را به‌خوبی مدیریت کنند تا یکپارچگی قرارداد حفظ شود و از رفتار غیرمنتظره جلوگیری شود.
- در استفاده از `call` برای فراخوانی‌های خارجی، حلقه‌ها و پیمایش‌ها، دقت داشته باشید تا از مصرف بیش از حد گاز که می‌تواند منجر به شکست تراکنش‌ها یا هزینه‌های غیرمنتظره شود، جلوگیری کنید.
- از اعطای دسترسی بیش از حد به یک نقش خاص در مجوزهای قرارداد خودداری کنید. به‌جای آن، دسترسی‌ها را به‌صورت منطقی تقسیم کنید و از کیف‌پول‌های چند امضایی برای مدیریت نقش‌هایی با مجوزهای حساس استفاده کنید تا از دست دادن مجوزها به دلیل افشای کلید خصوصی جلوگیری شود.

### مثال (قرارداد اصلاح شده):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    // Use a safer approach to transfer funds, like transfer, which has a fixed gas stipend.
    // This avoids using call and prevents issues with malicious fallback functions.
    function claimThrone() external payable {
        require(msg.value > balance, "Need to pay more to become the king");

        address previousKing = king;
        uint256 previousBalance = balance;

        // Update the state before transferring Ether to prevent reentrancy issues.
        king = msg.sender;
        balance = msg.value;

        // Use transfer instead of call to ensure the transaction doesn't fail due to a malicious fallback.
        payable(previousKing).transfer(previousBalance);
    }
}
```