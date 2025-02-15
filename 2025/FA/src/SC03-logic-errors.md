## SC03:2025 - خطاهای منطقی (Logic Errors)

### توضیحات: 
خطاهای منطقی که به‌عنوان Business Logic Vulnerabilities هم شناخته می‌شوند، مشکلاتی در Smart Contract‌ها هستند که باعث می‌شوند رفتار واقعی قرارداد با چیزی که انتظار می‌رود، یکی نباشد. این خطاها می‌توانند در بخش‌های مختلفی دیده شوند، مثل اشتباه در توزیع پاداش، نقص در مکانیزم Token Minting، یا محاسبات نادرست در فرآیند وام‌دهی و وام‌گیری. چنین باگ‌هایی معمولاً به‌سادگی دیده نمی‌شوند و ممکن است تا زمانی که اکسپلویت شوند، پنهان بمانند.

#### مثال هایی از خطاهای منطقی:
1. **توزیع نادرست پاداش:** اشتباه در محاسبه و تقسیم پاداش بین ذی‌نفعان که منجر به تخصیص ناعادلانه می‌شود.
2. **ایراد در مکانیزم مینت کردن توکن ها:** منطق نادرست یا کنترل‌نشده‌ای که امکان تولید بی‌نهایت یا ناخواسته‌ی توکن را فراهم می‌کند.
3. **عدم تعادل در استخر وام‌دهی:** ثبت نادرست واریزها و برداشت‌ها که باعث ایجاد ناهماهنگی در موجودی استخر می‌شود.

### مثال (قرارداد آسیب پذیر):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");

        // Faulty calculation: Incorrectly reducing the user's balance without updating the total lending pool
        userBalances[msg.sender] -= amount;

        // This should update the total lending pool, but it's omitted here.

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        // Faulty minting logic: Reward amount not validated
        userBalances[to] += rewardAmount;
    }
}
```

### شدت آسیب پذیری:
- خطاهای منطقی می‌توانند باعث رفتار غیرمنتظره Smart Contract شوند یا حتی آن را کاملاً از کار بیندازند. این مشکلات ممکن است منجر به موارد زیر شوند:

  - **از دست رفتن دارایی‌ها:** توزیع نادرست پاداش یا عدم تعادل در استخرها که باعث خالی شدن موجودی قرارداد می‌شود.
  - **ایجاد بیش از حد توکن:** افزایش غیرمجاز عرضه‌ی توکن که اعتماد و ارزش آن را از بین می‌برد.
  - **اختلال در عملکرد:** ناتوانی قرارداد در اجرای وظایف مورد انتظار.
- این پیامدها می‌توانند ضررهای مالی و عملیاتی جدی برای کاربران و  به همراه داشته باشند.

### توصیه های امنیتی:
- همیشه کد خود را با نوشتن تست کیس‌های جامع بررسی کنید تا تمامی سناریوهای ممکن در Business Logic پوشش داده شوند.
- کد را به‌دقت بازبینی و حسابرسی کنید تا خطاهای منطقی احتمالی شناسایی و برطرف شوند.
- رفتار مورد انتظار هر تابع و ماژول را مستند کنید و آن را با پیاده‌سازی واقعی مقایسه کنید تا مطمئن شوید که تطابق دارند.
- مکانیزم‌های کنترلی را پیاده‌سازی کنید، از جمله:
  -  استفاده از کتابخانه Safe Math  برای جلوگیری از خطاهای محاسباتی.
  - جرای کنترل‌های امنیتی و محدودیت‌های لازم برای Token Minting.
  - استفاده از الگوریتم‌های شفاف و قابل حسابرسی برای توزیع پاداش..

### مثال (ورژن اصلاح شده):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LogicErrors {
    mapping(address => uint256) public userBalances;
    uint256 public totalLendingPool;

    function deposit() public payable {
        userBalances[msg.sender] += msg.value;
        totalLendingPool += msg.value;
    }

    function withdraw(uint256 amount) public {
        require(userBalances[msg.sender] >= amount, "Insufficient balance");

        // Correctly reducing the user's balance and updating the total lending pool
        userBalances[msg.sender] -= amount;
        totalLendingPool -= amount;

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        require(rewardAmount > 0, "Reward amount must be positive");

        // Safeguarded minting logic
        userBalances[to] += rewardAmount;
    }
}
```

### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات منطق تجاری شده‌اند:


1. [Level Finance Hack](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO Hack](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)