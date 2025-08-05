## SC06:2025 - فراخوانی های خارجی چک نشده (Unchecked External Calls)

### توضیحات:
این آسیب‌پذیری زمانی رخ می‌دهد که یک قرارداد هوشمند در اتریوم یک فراخوانی خارجی به قرارداد یا آدرس دیگری ارسال می‌کند بدون اینکه نتیجه‌ی آن فراخوانی را بررسی کند. در اتریوم، زمانی که یک قرارداد، قراردادی دیگر را فراخوانی می‌کند، ممکن است قرارداد فراخوانی‌شده بدون ایجاد استثنا (Exception) با شکست مواجه شود. اگر قرارداد فراخوانی‌کننده نتیجه‌ی این فراخوانی را بررسی نکند، ممکن است به اشتباه فرض کند که عملیات موفق بوده است، حتی اگر شکست خورده باشد. این موضوع می‌تواند منجر به ناسازگاری در حالت (state) قرارداد و ایجاد آسیب‌پذیری‌هایی شود که مهاجمان می‌توانند از آن سوءاستفاده کنند.

### مثال (قرارداد آسیب پذیر):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        callee.delegatecall(_data);
    }
}
```
### شدت آسیب‌پذیری:
- این نوع آسیب‌پذیری ها می‌توانند منجر به ناموفق شدن تراکنش‌ها شوند، به‌طوری که عملیات مورد نظر با موفقیت انجام نمی‌شود. این موضوع ممکن است باعث از دست دادن سرمایه شود، زیرا قرارداد به اشتباه فرض می‌کند که انتقال با موفقیت انجام شده است. همچنین این وضعیت می‌تواند منجر به نادرست شدن حالت قرارداد شود و آسیب‌پذیری‌های دیگری در منطق قرارداد ایجاد کند که مهاجمان می‌توانند از آن بهره‌برداری کنند.


### راهکارهای امنیتی:
- در صورت امکان از transfer() به‌جای send() استفاده کنید، زیرا transfer() در صورت شکست فراخوانی خارجی تراکنش را معکوس می‌کند.
- همیشه نتیجه‌ی فراخوانی‌های send() یا call() را بررسی کنید تا در صورت بازگشت false به درستی مدیریت شود.

### مثال (قرار داد اصلاح شده):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_CheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // Ensure that delegatecall succeeds
        (bool success, ) = callee.delegatecall(_data);
        require(success, "Delegatecall failed");  // Check the return value to handle failure
    }
}
```

### مثال‌هایی از قراردادهای هوشمند که قربانی حملات Unchecked External Calls شده‌اند:
1. [Punk Protocol Hack](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)