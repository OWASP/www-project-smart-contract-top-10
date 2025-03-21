## SC01:2025 -  آسیب‌پذیری: کنترل دسترسی غیرمجاز (Improper Access Control)

### توضیحات:
 این آسیب‌پذیری یک مشکل امنیتی است که به کاربران غیرمجاز اجازه می‌دهد به داده‌ها یا توابع قرارداد هوشمند دسترسی پیدا کرده یا آن‌ها را تغییر دهند. این نوع آسیب‌پذیری زمانی رخ می‌دهد که کد قرارداد به درستی دسترسی‌ها را بر اساس سطوح مجوز کاربر محدود نمی‌کند. کنترل دسترسی در قراردادهای هوشمند می‌تواند به بخش‌های حیاتی مانند حاکمیت و منطق مهمی مانند مینت کردن توکن‌ها، رأی‌گیری در مورد پیشنهادات، برداشت وجوه، توقف یا به‌روزرسانی قراردادها و تغییر مالکیت مرتبط باشد.

### مثال (قرارداد آسیب پذیر):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_AccessControl {
    mapping(address => uint256) public balances;

    // Burn function with no access control
    function burn(address account, uint256 amount) public {
        _burn(account, amount);
    }
}
```
### شدت آسیب‌پذیری:

- مهاجمان می‌توانند به توابع و داده‌های حیاتی درون قرارداد دسترسی غیرمجاز پیدا کنند که این موضوع به اعتبار و امنیت قرارداد آسیب می‌زند.

- این آسیب‌پذیری‌ها می‌توانند منجر به سرقت وجوه یا دارایی‌های تحت کنترل قرارداد شوند و خسارت مالی قابل توجهی به کاربران و ذینفعان وارد کنند.


### توصیه های امنیتی:

- مطمعن شوید که توابع اولیه‌ (initialization functions) فقط یک‌بار و به‌طور انحصاری توسط بخش‌های مجاز فراخوانی شوند.


- از الگوهای کنترل دسترسی معتبر مانند **Ownable** یا **RBAC** (کنترل دسترسی مبتنی بر نقش) در قراردادهای خود استفاده کنید تا مجوزها را مدیریت کرده و اطمینان حاصل کنید که تنها کاربران مجاز به توابع خاصی دسترسی دارند. این کار می‌تواند با اضافه کردن مدیفایرهای کنترل دسترسی مناسب، مانند `onlyOwner` یا نقش‌های سفارشی به توابع حساس انجام شود.



### مثال (قرارداد اصلاح شده):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Import the Ownable contract from OpenZeppelin to manage ownership
import "@openzeppelin/contracts/access/Ownable.sol";

contract Solidity_AccessControl is Ownable {
    mapping(address => uint256) public balances;

    // Burn function with proper access control, only accessible by the contract owner
    function burn(address account, uint256 amount) public onlyOwner {
        _burn(account, amount);
    }
}
```

### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات کنترل دسترسی نامناسب شدند:
1. [HospoWise Hack](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT Hack](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)