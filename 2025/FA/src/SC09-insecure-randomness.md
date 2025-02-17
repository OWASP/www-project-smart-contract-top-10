## SC09:2025 - ایجاد اعداد رندوم نا امن (Insecure Randomness)

### توضیحات:
تولیدکننده‌های اعداد تصادفی برای برنامه‌هایی مانند شرط‌بندی، انتخاب برنده بازی و تولید اعداد تصادفی اولیه بسیار مهم هستند. در شبکه اتریوم، تولید اعداد تصادفی به دلیل ماهیت تعیین‌شده آن یک چالش محسوب می‌شود. از آنجا که Solidity قادر به تولید اعداد تصادفی واقعی نیست، به عوامل شبه‌تصادفی تکیه می‌کند. علاوه بر این، محاسبات پیچیده در Solidity از نظر مصرف گس هزینه‌بر هستند.


*مکانیسم‌های ناامن برای تولید اعداد تصادفی در Solidity: توسعه‌دهندگان اغلب از روش‌های مرتبط با بلاک برای تولید اعداد تصادفی استفاده می‌کنند، از جمله:*
  - block.timestamp: زمان‌بندی بلاک فعلی
  - blockhash(uint blockNumber): هش یک بلاک مشخص (تنها برای ۲۵۶ بلاک اخیر)
  - block.difficulty: سختی شبکه فعلی
  - block.number: شماره بلاک فعلی
  - block.coinbase: آدرس ماینر بلاک فعلی
    
این روش‌ها ناامن هستند زیرا ماینرها می‌توانند این مقادیر را دستکاری کنند و در نتیجه منطق قرارداد را تحت تأثیر قرار دهند.

### مثال (قرارداد آسیب پذیر):
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_InsecureRandomness {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // Using insecure mechanisms for random number generation
            ) 
        );

        if (_guess == answer) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```
### شدت آسیب پذیری:
- تصادفی‌سازی ناامن می‌تواند توسط مهاجمان برای کسب مزیت ناعادلانه در بازی‌ها، قرعه‌کشی‌ها و سایر قراردادهایی که به تولید اعداد تصادفی وابسته هستند، مورد سوءاستفاده قرار گیرد. مهاجمان با پیش‌بینی یا دستکاری نتایج به‌ظاهر تصادفی، می‌توانند نتایج را به نفع خود تغییر دهند. این امر می‌تواند به پیروزی‌های ناعادلانه، زیان‌های مالی برای سایر شرکت‌کنندگان و کاهش اعتماد به یکپارچگی و انصاف قرارداد هوشمند منجر شود.

### توصیه های امنیتی:
- استفاده از اوراکل‌ها (Oraclize) به عنوان منابع خارجی تصادفی‌سازی. باید دقت کرد که به اوراکل اطمینان کامل شود. استفاده از چندین اوراکل نیز می‌تواند کمک کند.
  
- استفاده از طرح‌های تعهدی(Commitment Schemes): یک ابتدایی رمزنگاری که از روش commit-reveal (تعهد-افشا) پیروی می‌کند. این روش در مواردی مانند پرتاب سکه، اثبات‌های بدون دانش (zero-knowledge proofs)، و محاسبات ایمن کاربرد دارد. مانند: RANDAO.
  
- استفاده از Chainlink VRF — یک تولیدکننده اعداد تصادفی اثبات‌شده و قابل‌تأیید است که به قراردادهای هوشمند اجازه دسترسی به مقادیر تصادفی را می‌دهد بدون آنکه امنیت یا کارایی به خطر بیافتد.
- 
- الگوریتم Signidice — برای تولید اعداد شبه‌تصادفی در برنامه‌هایی که شامل دو طرف هستند با استفاده از امضاهای رمزنگاری مناسب است.
  
- استفاده از هش بلاک بیت‌کوین — اوراکل‌هایی مانند BTCRelay که به عنوان پلی بین اتریوم و بیت‌کوین عمل می‌کنند. قراردادهای اتریوم می‌توانند از بلاک‌های آینده بیت‌کوین به عنوان منبعی برای تصادفی‌سازی استفاده کنند. باید توجه داشت که این روش در برابر مشکلات انگیزشی ماینرها ایمن نیست و باید با احتیاط پیاده‌سازی شود.


### مثال (قرارداد اصلاح شده):

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract Solidity_InsecureRandomness is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;

    constructor(address _vrfCoordinator, address _linkToken, bytes32 _keyHash, uint256 _fee) 
        VRFConsumerBase(_vrfCoordinator, _linkToken) 
    {
        keyHash = _keyHash;
        fee = _fee;
    }

    function requestRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "Not enough LINK");
        return requestRandomness(keyHash, fee);
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        randomResult = randomness;
    }

    function guess(uint256 _guess) public {
        require(randomResult > 0, "Random number not generated yet");
        if (_guess == randomResult) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Failed to send Ether");
        }
    }
}
```

### مثال‌هایی از قراردادهای هوشمندی که قربانی حملات تصادفی‌سازی ناامن شده‌اند:

1. [Roast Football Hack](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [FFIST Hack](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)