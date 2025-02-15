# SC04:2025 - عدم اعتبارسنجی ورودی‌ها (Lack of Input Validation)

## توضیحات:
اعتبارسنجی ورودی‌ها به قراردادهای هوشمند کمک می‌کند تا فقط داده‌های معتبر و مورد انتظار را پردازش کنند. اگر یک قرارداد ورودی‌های دریافتی را بررسی نکند، به‌طور ناخواسته در معرض خطراتی مانند دستکاری منطق، دسترسی غیرمجاز و رفتارهای غیرمنتظره قرار می‌گیرد.

برای مثال، اگر قرارداد فرض کند که ورودی‌های کاربران همیشه درست هستند و بدون بررسی آن‌ها را پردازش کند، مهاجمان می‌توانند از این ضعف سوءاستفاده کرده و داده‌های مخرب را وارد سیستم کنند. این موضوع امنیت و قابلیت اطمینان قرارداد هوشمند را به خطر می‌اندازد.

## مثال (قرارداد آسیب پذیر):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LackOfInputValidation {
    mapping(address => uint256) public balances;

    function setBalance(address user, uint256 amount) public {
        // The function allows anyone to set arbitrary balances for any user without validation.
        balances[user] = amount;
    }
}
```
### تاثیرات:
- مهاجمان می‌توانند با دستکاری ورودی‌ها، وجوه را تخلیه کنند، توکن‌ها را سرقت کنند یا آسیب‌های مالی دیگر ایجاد کنند.
- ورود داده‌های نادرست می‌تواند متغیرهای وضعیت قرارداد را خراب کرده و منجر به رفتار ناپایدار و ناامن شود.
- مهاجمان ممکن است از این ضعف برای انجام تراکنش‌ها یا عملیات غیرمجاز سوءاستفاده کنند، که می‌تواند هم کاربران و هم کل سیستم را تحت تأثیر قرار دهد.

### راهکارهای امنیتی:
- اطمینان حاصل کنید که ورودی‌ها با نوع داده‌ی مورد انتظار مطابقت دارند.
- بررسی کنید که مقادیر ورودی در محدوده‌ی مجاز قرار داشته باشند.
- مطمئن شوید که فقط کاربران یا قراردادهای مجاز بتوانند توابع خاصی را فراخوانی کنند..
- ساختار ورودی‌ها، مانند فرمت آدرس‌ها یا طول رشته‌ها را اعتبارسنجی کنید.
- همیشه در صورت نامعتبر بودن ورودی‌ها، اجرای قرارداد را متوقف کرده و پیام‌های خطای شفاف ارائه دهید.

### مثال (قرارداد اصلاح شده):
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
### نمونه‌هایی از قراردادهای هوشمندی که به دلیل نبود اعتبارسنجی ورودی مورد حمله قرار گرفتند:
1. [Convergence Finance](https://etherscan.io/address/0x2b083beaaC310CC5E190B1d2507038CcB03E7606#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/convergence-finance-hack-analysis-12e6acd9ea08)
2. [Socket Gateway](https://etherscan.io/address/0x3a23F943181408EAC424116Af7b7790c94Cb97a5#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/socket-gateway-hack-analysis-b0e9567f7d3e)