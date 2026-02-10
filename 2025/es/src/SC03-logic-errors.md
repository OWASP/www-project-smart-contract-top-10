## SC03:2025 - Errores lógicos

### Descripción: 
Los errores de lógica, también conocidos como vulnerabilidades de lógica de negocio, son fallos sutiles en los contratos inteligentes. Ocurren cuando el código del contrato no coincide con su comportamiento previsto. Estos errores pueden manifestarse de varias formas, como matemáticas defectuosas en la distribución de recompensas, mecanismos inadecuados de emisión de tokens, o cálculos incorrectos en la lógica de préstamos y empréstitos. Estas vulnerabilidades son escurridizas, se esconden dentro de la lógica del contrato y esperan a ser descubiertas.

#### Ejemplos de errores lógicos:
1. **Distribución defectuosa de recompensas:** Cálculos erróneos al dividir las recompensas entre las partes interesadas, lo que lleva a asignaciones injustas.
2. **Emisión incorrecta de fichas:** Lógica de emisión errónea o no comprobada que permite la generación infinita o no intencionada de tokens.
3. **Desequilibrios en el fondo de préstamos:** Seguimiento incorrecto de depósitos y retiradas, causando incoherencias en las reservas del fondo.

### Ejemplo (contrato vulnerable):
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

        // Cálculo erróneo: Reducción incorrecta del saldo del usuario sin actualizar el conjunto total de préstamos
        userBalances[msg.sender] -= importe;

        // Esto debería actualizar el saldo total de préstamos, pero se omite aquí.

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        // Lógica de emisión defectuosa: Importe de recompensa no validado
        userBalances[to] += rewardAmount;
    }
}
```

### Impacto:
- Los errores lógicos pueden provocar que un contrato inteligente se comporte de forma inesperada o incluso quede totalmente inutilizable. Estos errores pueden llevar a:
  - **Pérdida de fondos:** Distribución incorrecta de recompensas o desequilibrios en el pool que drenan los fondos del contrato.
  - **Excesiva emisión de tokens:** Inflación del suministro de tokens, socavando la confianza y el valor.
  - **Fallos operativos:** los contratos no cumplen las funciones previstas.
- Estas consecuencias pueden dar lugar a importantes pérdidas financieras y operativas para los usuarios y las partes interesadas.

### Solución:
- Valide siempre su código escribiendo casos de prueba exhaustivos que cubran todos los posibles escenarios de lógica empresarial.
- Realice revisiones y auditorías exhaustivas del código para identificar y corregir posibles errores lógicos.
- Documente el comportamiento previsto de cada función y módulo, y compárelo con la implementación real para garantizar la alineación.
- Implemente controles, como:
  - Bibliotecas matemáticas seguras para evitar errores de cálculo.
  - Controles y balances adecuados para la emisión de tokens.
  - Algoritmos de distribución de recompensas auditables.

### Ejemplo (Contrato Mejorado):
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
        require(userBalances[msg.sender] >= amount, "Saldo Insuficiente");

        // Reducir correctamente el saldo del usuario y actualizar el saldo total de préstamos
        userBalances[msg.sender] -= amount;
        totalLendingPool -= amount;

        payable(msg.sender).transfer(amount);
    }

    function mintReward(address to, uint256 rewardAmount) public {
        require(rewardAmount > 0, "El importe de la recompensa debe ser positivo");

          // Lógica de emisión protegida
        userBalances[to] += rewardAmount;
    }
}
```

### Ejemplos de Contratos Inteligentes Víctimas de Errores lógicos:
1. [Level Finance Hack](https://bscscan.com/address/0x9f00fbd6c095d2c542687ed5afb68d9c3fb2f464#code#F11#L165) : A Comprehensive [Hack Analysis](https://blog.solidityscan.com/level-finance-hack-analysis-16fda3996ecb)
2. [BNO Hack](https://bscscan.com/address/0xdca503449899d5649d32175a255a8835a03e4006#code) : Un completo [Hack Análisis](https://blog.solidityscan.com/bno-hack-analysis-15436d73e44e)