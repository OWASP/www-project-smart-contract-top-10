## SC08:2025 - Integer Overflow and Underflow

### توضیحات:
ماشین مجازی اتریوم (EVM) از نوع داده‌ای با اندازه ثابت برای integers استفاده می‌کند، این به این معناست که یک متغیر عددی فقط می‌تواند در محدوده مشخصی از اعداد قرار بگیرد.

1. Overflow (سرریز)

به عنوان مثال، اگر از یک نوع متغیر uint8 استفاده کنید، این متغیر فقط می‌تواند 8 بیت را درون خود ذخیره کند یعنی اعداد بین 0 تا 255. اگر شما سعی کنید عددی بزرگتر از 255 در این متغیر ذخیره کنید (مثل 256)، به جای خطا، مقدار آن دوباره به 0 برمی‌گردد. این اتفاق را overflow می‌گویند. همچنین اگر عددی بزرگتر نیز وارد کنید برای مثال 257 مقدار به 1 تغییر می‌کند

جایگذاری تصویر 

1. Underflow (زیرریز):

از سوی دیگر، اگر در یک uint8 که از 0 شروع می‌شود، مقدار 1 را کم کنید، به جای اینکه نتیجه به عدد منفی برسد، مقدار به 255 می‌رسد. این پدیده underflow نامیده می‌شود، یعنی هنگامی که مقدار از حداقل ممکن کمتر می‌شود و به حداکثر مقدار ممکن برمی‌گردد. 

1.مقایسه برای اعداد صحیح  :

برای متغیرهایی مثل int8، اگر به کوچک‌ترین مقدار ممکن یعنی منفی 128 (-128) برسیم و بخواهیم یک واحد از آن کم کنیم، مقدار به 127 می‌رسد. این به این دلیل است که متغیرهای علامت‌دار، پس از رسیدن به بزرگ‌ترین مقدار منفی، به مقدار مثبت بزرگ‌ترین مقدار برمی‌گردند.


**نکته مهم :**
در سالیدیتی `0.8.0` و ورژن بالاتر, کامپایلر به صورت خودکار overflow و underflow را بررسی می‌کند و در صورتی که این اتفاق درحال رخ دادن باشد تراکنش را برمی‌گرداند(revert).
همچنین در این ورژن از سالیدیتی کلمه کلیدی `unchecked` تعریف شده است که به توسعه دهندگان این اجازه را می‌هد که بدون بررسی خودکار بخشی از کد اجرا شود. درواقع استفاده از `unchecked` زمانی کاربردی  می‌باشد که :
- موضوع overflow و underflow مشکلی ایجاد نمی‌کنند.
- رفتار wraparound (یعنی چرخش مقادیر مثل قبل) مورد نیاز است.



### مثال (قرارداد آسیب پذیر):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.17;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() public {
        balance = 255; // Maximum value of uint8
    }

    // Increments the balance by a given value
    function increment(uint8 value) public {
        balance += value; // Vulnerable to overflow
    }

    // Decrements the balance by a given value
    function decrement(uint8 value) public {
        balance -= value; // Vulnerable to underflow
    }
}

```
### شدت آسیب پذیری:
- مهاجم می‌‌تواند با سوءاستفاده از این آسیب‌پذیری ها، موجودی حساب یا تعداد توکن هارا دستکاری و افزایش دهد و در نتیجه، بتواند بیش از دارایی واقعی خود برداشت انجام دهد.
- همچنین مهاجم ممکن است منطق قرارداد را تغییر دهد که منجر به اقدامات مانند سرقت دارایی ها یا مینت کردن تعدادی زیادی توکن شود.


### توصیه های امنیتی:
- ساده ترین و بهترین روش استفاده از ورژن 0.8.0 یا بالاتر سالیدیتی است، چراکه این نسخه به طور خودکار این موضوع را بررسی و در صورت وقوع، تراکنش را برمی‌گرداند.
- استفاده از آخرین نسخه کتابخانه SafeMath : این کتابخانه یکی از پروژه هایی که OpenZeppelin ایجاد کرده که می‌تواند برای جلوگیری از این آسیب پذیری استفاده شود.این کتابخانه توابعی مانند `mul`, `sub()`,`add()` و غیره را فراهم می‌کند که عملیات حسابی پایه را انجام داده و در صورت وقعود این آسیب پذیری به طور خودکار تراکنش را بر‌میگرداند.

### مثال (قرارداد اصلاح شده):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() {
        balance = 255; // Maximum value of uint8
    }

    // Increments the balance by a given value
    function increment(uint8 value) public {
        balance += value; // Solidity 0.8.x automatically checks for overflow
    }

    // Decrements the balance by a given value
    function decrement(uint8 value) public {
        require(balance >= value, "Underflow detected");
        balance -= value;
    }
}
```

### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات Integer Overflow و Underflow شدند:
1. [PoWH Coin Ponzi Scheme](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)