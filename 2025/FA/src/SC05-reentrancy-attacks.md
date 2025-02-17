## SC05:2025 - حملات بازگشت‌پذیر (Reentrancy)

### توضیحات:

حمله بازگشت مجدد یا Reentrancy attack درواقع نوعی آسیب پذیری در قراردادهای هوشمند هست به شکلی که با استفاده از تابعی به نام Fallback برای فراخوانی مجدد استفاده می‌کند. این امر می‌تواند منجر به از دست دادن دارایی ها در قرارداد‌های هوشمند شود. در زبان برنامه نویسی سالیدیتی تابعی به نام Fallback  وجود دارد که پارامتی نداشته و به صورت دلخواه برنامه نویس می‌تواند درون کد قراردهد.

### مثال (قرار داد ‌آسیب پذیر): 
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

        // Vulnerability: Ether is sent before updating the user's balance, allowing reentrancy.
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // Update balance after sending Ether
        balances[msg.sender] = 0;
    }
}
```
### شدت تاثیرگذاری آسیب‌پذیری‌:

- **خروج غیرمجاز وجوه:** مهاجمان با بهره‌گیری از این آسیب‌پذیری می‌توانند بیشتر از میزان مجاز وجوه برداشت کنند و حتی موجودی قرارداد را کاملاً تخلیه کنند.
- **فراخوانی غیرمجاز توابع:** مهاجم می‌تواند توابعی را به صورت غیرمجاز فراخوانی کند که این می‌تواند منجر به انجام عملیات ناخواسته در قرارداد یا سیستم‌های مرتبط شود.

### توصیه ها:

- **به‌روزرسانی موجودی‌ها قبل از فراخوانی کد خارجی:** همیشه اطمینان حاصل کنید که هر تغییر وضعیت داخلی قبل از فراخوانی قراردادهای خارجی انجام می‌شود. به عبارت دیگر، ابتدا موجودی‌ها یا کد داخلی را به‌روزرسانی کنید و سپس کد خارجی را فراخوانی کنید.
- **استفاده از Re-entrancy Guard:** از `ReentrancyGuard` یا محافظ‌های مشابه برای جلوگیری از بازگشت مجدد استفاده کنید، مانند کتابخانه‌های OpenZeppelin.

### مثال (قرارداد اصلاح شده):

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

        // Fix: Update the user's balance before sending Ether
        balances[msg.sender] = 0;

        // Then send Ether
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات بازگشت پذیر شده‌اند:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)