## SC09:2026 - 정수 오버플로 및 언더플로

#### 설명

정수 오버플로 및 언더플로는 산술 연산이 피연산자 타입의 표현 가능한 범위를 벗어나는 값을 생성하는 상황을 말합니다. Solidity 0.8+에서는 산술이 **기본적으로 검사**되며 오버플로/언더플로 시 되돌립니다. 그러나 명시적인 `unchecked` 블록, 어셈블리, 또는 커스텀 라이브러리는 이러한 검사를 비활성화할 수 있습니다. 비EVM 플랫폼(예: Move, Sui, Solana, Rust 기반 체인)에서는 기본 오버플로 의미론이 다릅니다—일부는 조용히 래핑하고, 일부는 중단합니다—그리고 잘못된 가정이나 결함 있는 커스텀 검사는 래핑된 값, 잘못 계산된 잔액, 깨진 불변량으로 이어질 수 있습니다.

이는 산술을 수행하는 모든 컨트랙트 유형에 영향을 미칩니다: DeFi(풀 불변량, 잔액, 이자, 지분), NFT(공급, 토큰 ID), 브리지(금액, 시퀀스 번호), 그리고 크거나 사용자 제어 숫자 입력을 포함하는 모든 로직. 오버플로/언더플로가 경제적 불변량을 깨거나(예: AMM의 k = x * y) 잔액 조작을 가능하게 할 때 영향이 특히 심각합니다.

주요 점검 영역:

- **EVM/Solidity** (`unchecked` 사용, 어셈블리, 0.8 이전 코드베이스)
- **비EVM 체인** (Move, Sui, Aptos, Solana 등) 및 기본 오버플로 의미론
- **곱셈 및 지수화** (큰 피연산자로 오버플로 위험 높음)
- **뺄셈 및 감소** (빼는 수가 원래 값보다 클 때 언더플로)
- **캐스팅 및 타입 변환** (uint256을 uint128로 다운캐스팅 등)

공격자가 악용하는 방식:

- 오버플로/언더플로가 불가능하다고 가정되지만 엣지 케이스가 존재하는 Solidity의 **unchecked 블록**
- 조용한 래핑 또는 커스텀 검사가 우회될 수 있는 **비EVM 의미론**
- 곱셈 또는 덧셈 체인에서 오버플로를 유발하는 **크거나 조작된 입력**
- 순진한 검사를 통과하는 작은 k를 생성하는 오버플로 같은 **불변량 깨는 값**

### 예시 1: Solidity 0.8 이전 오버플로 (EVM)

Solidity **0.8.0 이전** 버전에서는 산술 오버플로와 언더플로가 **조용히** 발생했습니다—되돌리기 없음, 오류 없음. 값이 래핑되었습니다 (예: `uint8` 255 + 1 = 0). Solidity 0.8.0+는 기본적으로 이를 수정합니다; 오버플로/언더플로는 명시적으로 `unchecked`로 래핑되지 않는 한 되돌립니다.

**취약 (Solidity 0.7.x):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

contract VulnerableToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        // 언더플로: balances[msg.sender] < amount이면 큰 값으로 래핑
        balances[msg.sender] -= amount;  // 조용한 언더플로!
        balances[to] += amount;          // 조용한 오버플로 가능
    }
}
```

**수정 (옵션 A—0.7.x용 SafeMath):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.7.6;

import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract SafeToken {
    using SafeMath for uint256;
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        balances[msg.sender] = balances[msg.sender].sub(amount);  // 언더플로 시 되돌림
        balances[to] = balances[to].add(amount);                   // 오버플로 시 되돌림
    }
}
```

**수정 (옵션 B—Solidity 0.8+로 업그레이드):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract SafeToken {
    mapping(address => uint256) public balances;

    function transfer(address to, uint256 amount) external {
        balances[msg.sender] -= amount;  // 언더플로 시 되돌림 (기본적으로 검사됨)
        balances[to] += amount;          // 오버플로 시 되돌림
    }
}
```

**핵심 교훈:** Solidity 0.8.0+는 내장 오버플로/언더플로 검사를 제공합니다. 강력한 보장이 있을 때만 `unchecked`를 사용하고; 그렇지 않으면 검사된 산술을 선호합니다.

---

### 예시 2: Cetus Protocol—Sui/Move (비EVM)

**2025년 5월 22일**, Cetus Protocol(Sui 최대 DEX)이 `integer-mate` 라이브러리의 결함 있는 오버플로 검사로 인해 약 $2억 2,300만을 잃었습니다. **Move**에서는 덧셈과 곱셈이 오버플로 시 중단되지만, **왼쪽 시프트(`<<`)는 중단되지 않습니다**—조용히 절사됩니다. 프로토콜은 `<< 64` 시프트를 보호하기 위해 커스텀 `checked_shlw`를 사용했습니다; 가드가 잘못되었습니다.

**근본 원인:** `math_u256.move`의 `checked_shlw`가 잘못된 임계값을 사용했습니다. 상위 64비트에 0이 아닌 비트가 있는 값을 거부해야 합니다 (즉, `n >= 1 << 192`). 구현은 대신 `n > (0xFFFFFFFFFFFFFFFF << 192)`를 사용했는데, 이는 잘못되어 2^192 이상의 값이 통과하도록 허용한 다음 `n << 64`에서 오버플로가 발생했습니다.

**취약한 `checked_shlw` (integer-mate, Sui Move):**

```move
// integer-mate/sui/sources/math_u256.move
// 취약: 잘못된 오버플로 임계값
public fun checked_shlw(n: u256): (u256, bool) {
    let mask = 0xFFFFFFFFFFFFFFFF << 192;  // 잘못됨! 잘못된 임계값 생성
    if (n > mask) {
        (0, true)   // 오버플로 신호를 보내야 함
    } else {
        ((n << 64), false)  // n >= 2^192에서 오버플로 발생—Move가 조용히 절사
    }
}
```

**수정된 `checked_shlw`:**

```move
// 수정: 올바른 오버플로 검사—상위 64비트에 비트가 있으면 거부
public fun checked_shlw(n: u256): (u256, bool) {
    // 올바름: 64비트 왼쪽 시프트는 n >= 2^192이면 오버플로
    if (n >= 1 << 192) {
        (0, true)   // 오버플로—중단 경로
    } else {
        ((n << 64), false)
    }
}
```

**사용된 위치:** `clmm_math.move`의 CLMM 함수 `get_delta_a`가 유동성 포지션에 필요한 토큰 A를 계산했습니다. 다음을 호출했습니다:

```move
let (numerator, overflowing) = math_u256::checked_shlw(
    full_math_u128::full_mul(liquidity, sqrt_price_diff)
);
assert!(!overflowing);  // 결함 있는 검사로 인해 어설션이 잘못 통과됨
```

`liquidity ≈ 2^113`이고 `sqrt_price_diff ≈ 2^79`이면 곱은 `≈ 2^192 + ε`입니다. 결함 있는 `checked_shlw`가 이를 통과시켰습니다; `n << 64`가 오버플로되어 작은 값으로 절사되었습니다. 그러면 프로토콜은 거대한 유동성(~10^37 단위)을 발행하는 데 **1 단위**의 토큰 A만 필요하다고 계산하여 소진을 가능하게 했습니다.

**공격 흐름 (단순화):** 플래시 스왑 → 좁은 틱 포지션 열기 → 조작된 유동성 파라미터로 `add_liquidity` 호출 → 과소 청구 (1 토큰)되면서 거대한 유동성 적립 → 유동성 제거 → 풀 소진 → 플래시 스왑 상환.

---

### 예시 3: Solidity 0.8+에서 `unchecked` (명시적 옵트아웃)

Solidity 0.8+에서도 `unchecked`는 검사를 비활성화합니다. 오버플로/언더플로가 증명 가능하게 불가능할 때만 사용합니다:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract UncheckedExample {
    function bad(uint256 x, uint256 y) external pure returns (uint256) {
        unchecked {
            return x * y;  // 오버플로 가능; 되돌리기 없음
        }
    }

    function good(uint256 x) external pure returns (uint256) {
        unchecked {
            return x - 1;  // 호출자가 x >= 1을 보장하는 경우에만 안전
        }
    }
}
```

공식적 추론이나 안전성을 증명하는 테스트가 없는 한 사용자 제어 또는 무제한 입력에 **`unchecked` 사용을 피합니다**.

---

### 2025년 사례 연구: Cetus (2025년 5월, $2억 2,300만 손실)

Sui의 Cetus Protocol이 공유 `integer-mate` 라이브러리의 결함 있는 `checked_shlw`를 통해 익스플로잇되었습니다. 이 함수는 CLMM 유동성 계산 중 64비트 왼쪽 시프트 시 u256 오버플로를 방지하기 위한 것이었습니다. 오버플로 검사가 잘못된 임계값(`1 << 192` 대신 `0xFFFFFFFFFFFFFFFF << 192`)을 사용하여 2^192 이상의 값이 통과하도록 허용했습니다. Move에서 왼쪽 시프트는 오버플로 시 중단되지 않습니다—절사됩니다. 절사된 분자로 인해 `get_delta_a`가 거대한 유동성을 발행하는 데 1 토큰만 필요하다고 반환했습니다. 공격자는 플래시 스왑을 사용하여 여러 풀에 걸쳐 이를 반복하여 약 $2억 2,300만을 소진했습니다. 핵심 교훈: (1) Move 시프트 연산은 오버플로 시 중단되지 않습니다; (2) 커스텀 오버플로 가드는 엄격하게 검증되어야 합니다; (3) 공유 수학 라이브러리는 고위험이며 공식 분석이 필요합니다.  
- [https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/](https://dedaub.com/blog/the-cetus-amm-200m-hack-how-a-flawed-overflow-check-led-to-catastrophic-loss/)
- [https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis](https://www.cyfrin.io/blog/inside-the-223m-cetus-exploit-root-cause-and-impact-analysis)
- [https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025](https://www.halborn.com/blog/post/explained-the-cetus-hack-may-2025)
- [https://blog.verichains.io/p/cetus-protocol-hacked-analysis](https://blog.verichains.io/p/cetus-protocol-hacked-analysis)

### 모범 사례 및 완화 방법

- Solidity/EVM에서:
  - 강력한 이유와 안전성을 증명하는 테스트가 없는 한 **`unchecked` 산술 사용 회피**.
  - 중요한 불변량에 명시적 검사와 커스텀 에러 사용.
  - 잘 검토된 수학 라이브러리 선호 (고정소수점, 지수화 등).
- 비EVM 환경(예: Move, Rust 기반 체인)에서:
  - 언어의 **기본 오버플로 의미론** 이해.
  - 가용한 경우 안전한 산술 구조 또는 라이브러리 사용.
  - 중요한 산술 주변에 **어설션 및 불변량 추가**.
- **극단적인 값 범위**로 테스트:
  - 모든 숫자 타입의 최솟값 및 최댓값.
  - 오버플로/언더플로가 발생하기 쉬운 경계 근처의 엣지 케이스를 대상으로 하는 퍼즈 테스트.

