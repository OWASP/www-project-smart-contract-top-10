## آسیب‌پذیری: خطاهای منطقی (Logic Errors)

### توضیحات: 
خطاهای منطقی، که به‌عنوان آسیب‌پذیری‌های بیزنس لاجیک نیز شناخته می‌شوند، مشکلات جزئی و پنهانی در قراردادهای هوشمند هستند. این خطاها زمانی رخ می‌دهند که کد قرارداد با رفتار مورد انتظار آن مطابقت ندارد. چنین خطاهایی پیچیده و مبهم بوده و ممکن است در منطق قرارداد پنهان شوند و در آینده کشف گردند.

### مثال :
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LendingPlatform {
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
}
```
### شدت آسیب‌پذیری:
- خطاهای منطقی می‌توانند باعث شوند که یک قرارداد هوشمند رفتار غیرمنتظره‌ای از خود نشان دهد یا حتی به‌طور کامل غیرقابل استفاده شود. این خطاها می‌توانند منجر به از دست رفتن وجوه، توزیع نادرست توکن‌ها یا سایر نتایج نامطلوب شوند و پیامدهای مالی و عملیاتی قابل توجهی برای کاربران و ذی‌نفعان ایجاد کنند.
  
### راهکارهای امنیتی:
- همیشه کد خود را با نوشتن تست‌های جامع که تمام منطق تجاری ممکن را پوشش می‌دهند، اعتبارسنجی کنید.

- بررسی دقیق کد و انجام ممیزی‌ها برای شناسایی و رفع خطاهای منطقی احتمالی را انجام دهید.

- رفتار مورد انتظار هر تابع و ماژول را مستند کنید و سپس آن را با پیاده‌سازی واقعی مقایسه کنید تا اطمینان حاصل شود که همه چیز مطابق انتظار است.


### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات منطق تجاری شده‌اند:
1. [Level Finance Hack](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO Hack](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)
