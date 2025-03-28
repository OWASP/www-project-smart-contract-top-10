## SC06:2025 Llamadas externas no verificadas

### Descripción:
Las llamadas externas no comprobadas se refieren a un fallo de seguridad en el que un contrato realiza una llamada externa a otro contrato o dirección sin comprobar adecuadamente el resultado de dicha llamada. En Ethereum, cuando un contrato llama a otro contrato, el contrato llamado puede fallar silenciosamente sin lanzar una excepción. Si el contrato que llama no comprueba el valor de retorno, podría asumir incorrectamente que la llamada fue exitosa, incluso si no lo fue. Esto puede llevar a inconsistencias en el estado del contrato y vulnerabilidades que los atacantes pueden explotar.

### Ejemplo (Contrato vulnerable):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.4.24;

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() public {
        owner = msg.sender;
    }

    function forward(address callee, bytes _data) public {
        require(callee.delegatecall(_data));
    }
}
```
### Impacto:
- Las llamadas externas no verificadas pueden dar lugar a transacciones fallidas, provocando que las operaciones previstas no se completen con éxito. Esto puede conducir a la pérdida de fondos, ya que el contrato puede proceder bajo la falsa suposición de que la transferencia se ha realizado correctamente. Además, puede crear un estado de contrato incorrecto, haciendo que el contrato sea vulnerable a más exploits e inconsistencias en su lógica.

### Solución:
- Siempre que sea posible, utilice transfer() en lugar de send(), ya que transfer() revierte la transacción si la llamada externa falla.
- Compruebe siempre el valor de retorno de las funciones send() o call() para asegurar un manejo adecuado si devuelven false.

### Ejemplo (Contrato Mejorado):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_UncheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // Asegurarse que delegatecall tiene éxito
        (bool éxito, ) = callee.delegatecall(_datos);
        require(success, «Delegatecall failed»); // Comprueba el valor de retorno para manejar el fallo
    }
}
```

### Ejemplos de Contratos Inteligentes que fueron Víctimas de Ataques de Llamadas Externas sin Control:
1. [Protocolo Punk](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) :Un completo [Análisis del Hackeo](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)