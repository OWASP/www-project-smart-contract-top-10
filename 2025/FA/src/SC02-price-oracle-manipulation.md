# SC02:2025 - دستکاری در Price Oracle
<span dir="rtl" align="right">


## توضیحات:
دستکاری در **Price Oracle** یکی از آسیب‌پذیری‌های جدی در **Smart Contract**‌هایی است که برای دریافت قیمت‌ها یا اطلاعات دیگر به **Oracles** (منابع داده‌ی خارجی) متکی هستند. در **DeFi**، اوراکل‌ها داده‌های دنیای واقعی، مانند قیمت دارایی‌ها، را به قراردادهای هوشمند ارائه می‌کنند.

اما اگر داده‌های ارسال‌شده توسط اوراکل دستکاری شوند، ممکن است منجر به رفتار نادرست قرارداد شود. مهاجمان می‌توانند با تغییر اطلاعاتی که اوراکل تأمین می‌کند، از آن سوءاستفاده کنند و پیامدهای مخربی ایجاد کنند، از جمله:


- برداشت‌های غیرمجاز  
- ایجاد لوریج بیش از حد  
- تخلیه‌ی استخرهای نقدینگی  

برای جلوگیری از این نوع حملات، استفاده از **مکانیزم‌های امنیتی و اعتبارسنجی مناسب** ضروری است.

</span>

## مثال (قرارداد آسیب‌پذیر):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0, "Price must be positive");

        // Vulnerability: No validation or protection against price manipulation
        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on manipulated price
        // If an attacker manipulates the oracle, they could borrow more than they should
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```

### تاثیرات آسیب پذیری:
- مهاجمان می‌توانند با دستکاری اوراکل، قیمت یک دارایی را به‌طور مصنوعی افزایش دهند و این امکان را پیدا کنند که بیش از حد مجاز، وام دریافت کنند.
- در مواردی که قیمت دستکاری‌شده باعث ارزیابی نادرست وثیقه شود، کاربران قانونی ممکن است به دلیل ارزش‌گذاری نادرست، نقد شوند.
- اگر یک اوراکل به خطر بیفتد، مهاجمان می‌توانند از داده‌های تغییر یافته برای تخلیه‌ی استخرهای نقدینگی قرارداد یا حتی منجر به ورشکستگی آن شوند.


### توصیه‌های امنیتی:
- داده‌ها را از **چندین اوراکل مستقل** تجمیع کنید تا خطر دستکاری توسط یک منبع خاص کاهش یابد.
- برای قیمت‌های دریافتی از اوراکل، **حداقل و حداکثر مقدار مجاز** تعیین کنید تا نوسانات شدید قیمت، منطق قرارداد را تحت تأثیر قرار ندهد.
- بین به‌روزرسانی‌های قیمت، **وقفه زمانی (Time Lock)** ایجاد کنید تا از تغییرات آنی که می‌تواند توسط مهاجمان سوءاستفاده شود، جلوگیری شود.
- از **اثبات‌های رمزنگاری‌شده** برای تأیید صحت داده‌های دریافت‌شده از اوراکل‌ها استفاده کنید، مانند الزام به امضای داده‌ها توسط منابع معتبر.


### مثال (قرارداد اصلاح شده):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // Set minimum acceptable price
    int public maxPrice = 2000; // Set maximum acceptable price

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // Borrow logic based on valid price
    }

    function repay(uint256 amount) public {
        // Repayment logic
    }
}
```
###  مثال‌هایی از قراردادهای هوشمندی که قربانی حملات دستکاری اوراکل شدند:
1. [Polter Finance Hack Analysis](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO Protocol](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)
