## SC06:2025 Llamadas externas no verificadas

### Descripción:
Las llamadas externas no comprobadas se refieren a un fallo de seguridad en el que un contrato realiza una llamada externa a otro contrato o dirección sin comprobar adecuadamente el resultado de dicha llamada. En Ethereum, cuando un contrato llama a otro contrato, el contrato llamado puede fallar silenciosamente sin lanzar una excepción. Si el contrato que llama no comprueba el valor de retorno, podría asumir incorrectamente que la llamada fue exitosa, incluso si no lo fue. Esto puede llevar a inconsistencias en el estado del contrato y vulnerabilidades que los atacantes pueden explotar.

### Ejemplo (Contrato vulnerable):
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
### Impacto:
- Las llamadas externas no verificadas pueden dar lugar a transacciones fallidas, provocando que las operaciones previstas no se completen con éxito. Esto puede conducir a la pérdida de fondos, ya que el contrato puede proceder bajo la falsa suposición de que la transferencia se ha realizado correctamente. Además, puede crear un estado de contrato incorrecto, haciendo que el contrato sea vulnerable a más exploits e inconsistencias en su lógica.

### Solución:
- Siempre que sea posible, utilice transfer() en lugar de send(), ya que transfer() revierte la transacción si la llamada externa falla.
- Compruebe siempre el valor de retorno de las funciones send() o call() para asegurar un manejo adecuado si devuelven false.

### Ejemplo (Contrato Mejorado):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0; 

contract Solidity_CheckedExternalCall {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function forward(address callee, bytes memory _data) public {
        // Asegurarse que delegatecall tiene éxito
        (bool success, ) = callee.delegatecall(_data);
        require(success, "Delegatecall failed"); // Comprueba el valor de retorno para manejar el fallo
    }
}
```

### Advertencias

Los dos contratos anteriores contienen debilidades más allá de un valor de retorno no verificado.

- La autenticación se delega al contrato llamado. El código del contrato llamado puede, o no, restringir el valor de `msg.sender`, por ejemplo, comparándolo con `owner`. Normalmente, la función `forward` debería realizar algún tipo de autenticación.
- `callee` es una dirección proporcionada por el usuario. Esto significa que se puede ejecutar código arbitrario en el contexto de este contrato, modificando por ejemplo `owner`. Esto es particularmente problemático, ya que `forward` no realiza autenticación.
- No se verifica si la dirección `callee` corresponde a un contrato. Si `callee` es una dirección sin código, esto pasará desapercibido, ya que `delegatecall` tendrá éxito. Normalmente, la función `forward` debería hacer verificaciones básicas, como confirmar que el tamaño del código del contrato llamado sea mayor que cero.

### Ejemplos de Contratos Inteligentes que fueron Víctimas de Ataques de Llamadas Externas sin Control:
1. [Protocolo Punk](https://github.com/PunkFinance/punk.protocol/blob/master/contracts/models/CompoundModel.sol) :Un completo [Análisis del Hackeo](https://blog.solidityscan.com/security-issues-with-delegate-calls-4ae64d775b76)