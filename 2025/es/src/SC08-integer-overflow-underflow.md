## SC08:2025 - Desbordamiento y Subdesbordamiento de Enteros  

### Descripción  
La Ethereum Virtual Machine (EVM) define tipos de datos de tamaño fijo para los enteros. Esto implica que el rango de números que una variable entera puede representar es finito. Por ejemplo, un `uint8` (entero sin signo de 8 bits, es decir, no negativo) solo puede almacenar valores entre 0 y 255. Intentar almacenar un valor mayor a 255 en un `uint8` provocará un **desbordamiento** (overflow). De manera similar, restar `1` a `0` generará 255, lo que se conoce como **subdesbordamiento** (underflow).  

Cuando una operación aritmética excede o cae por debajo del tamaño máximo o mínimo de un tipo, se produce un desbordamiento o subdesbordamiento. Para los enteros con signo, el resultado es distinto: si intentamos restar `1` a un `int8` cuyo valor es `-128`, el resultado será `127`. Esto ocurre porque los enteros con signo, que pueden representar valores negativos, se reinician cuando alcanzan el valor negativo más bajo.  

Ejemplos sencillos de este comportamiento incluyen:
- Funciones matemáticas periódicas (sumar `2` al argumento de `sin(x)` no altera el resultado).  
- Odómetros en automóviles, que se reinician a `000000` tras alcanzar su límite (`999999`).  

***Nota Importante:- 
A partir de Solidity `0.8.0`, el compilador maneja automáticamente la verificación de desbordamientos y subdesbordamientos en las operaciones aritméticas, revirtiendo la transacción si ocurren.  
Solidity `0.8.0` también introduce la palabra clave `unchecked`, que permite a los desarrolladores realizar operaciones aritméticas sin estas verificaciones automáticas, permitiendo explícitamente el desbordamiento sin revertir la transacción. Esto puede ser útil para optimizar el uso de gas en casos donde el desbordamiento no sea un problema o donde se desee este comportamiento, como en versiones anteriores de Solidity.***


### Ejemplo (Contrato Vulnerable):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.17;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() public {
        balance = 255; // Maximum value of uint8
    }

    // Incrementa el balance en un valor dado
    function increment(uint8 value) public {
        balance += value; // Vulnerable a desbordamiento
    }

    // Decrementa el balance en un valor dado
    function decrement(uint8 value) public {
        balance -= value; // Vulnerable a subdesbordamiento
    }
}

```
### Impacto
- Un atacante podría explotar estas vulnerabilidades para aumentar artificialmente saldos de cuentas o cantidades de tokens, permitiéndoles retirar más fondos de los que realmente poseen.
- Un atacante podría alterar la lógica del contrato y ejecutar acciones no autorizadas, como robar activos o emitir una cantidad excesiva de tokens.

### Remediación
- La forma más sencilla de prevenir estas vulnerabilidades es usar Solidity 0.8.0 o versiones posteriores, ya que manejan automáticamente los desbordamientos y subdesbordamientos.
- Utilizar librerías de seguridad, como SafeMath de OpenZeppelin, que han sido ampliamente auditadas por la comunidad de Ethereum. SafeMath proporciona funciones como add(), sub(), mul(), etc., que realizan operaciones aritméticas básicas y revierten automáticamente si ocurre un desbordamiento o subdesbordamiento.

### Ejemplo (Version Mejorada):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_OverflowUnderflow {
    uint8 public balance;

    constructor() {
        balance = 255; // Valor máximo de uint8
    }

    // Incrementa el balance en un valor dado
    function increment(uint8 value) public {
        balance += value; // Solidity 0.8.x verifica automáticamente el desbordamiento
    }

    // Decrementa el balance en un valor dado
    function decrement(uint8 value) public {
        require(balance >= value, "Subdesbordamiento detectado");
        balance -= value;
    }
}
```

### Ejemplos de Contratos Inteligentes Víctimas de Ataques de Desbordamiento y Subdesbordamiento de Enteros
1. [PoWH Coin Ponzi Scheme](https://etherscan.io/token/0xa7ca36f7273d4d38fc2aec5a454c497f86728a7a#code) : Un completo análsis del [Hack](https://blog.solidityscan.com/integer-overflow-and-underflow-in-smart-contracts-9598032b5a99)
2. [Poolz Finance](https://bscscan.com/address/0x8bfaa473a899439d8e07bf86a8c6ce5de42fe54b#code) : Un completo análisis del [Hack](https://blog.solidityscan.com/poolz-finance-hack-analysis-still-experiencing-overflow-fcf35ab8a6c5)