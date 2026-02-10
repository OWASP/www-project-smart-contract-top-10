## SC05:2025 - Reentrancy (Reentrada)

### Descripción:
Un ataque de reentrada explota la vulnerabilidad en contratos inteligentes cuando una función hace una llamada externa a otro contrato antes de actualizar su propio estado. Esto permite al contrato externo, posiblemente malicioso, volver a entrar en la función original y repetir ciertas acciones, como retiradas, utilizando el mismo estado. A través de este tipo de ataques, un atacante puede drenar todos los fondos de un contrato.

### Ejemplo (Contrato Vulnerable): 
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Vulnerabilidad: Se envía Ether antes de actualizar el saldo del usuario, permitiendo la reentrada.
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");

        // Actualiza saldo después de enviar Ether
        balances[msg.sender] = 0;
    }
}
```
### Impacto:
- La consecuencia más inmediata e impactante es el vaciado de fondos. Los atacantes explotan las vulnerabilidades para retirar más dinero del que tienen derecho, vaciando potencialmente el saldo del contrato por completo.
- Un atacante puede desencadenar llamadas a funciones no autorizadas. Esto puede llevar a que se ejecuten acciones no deseadas dentro del contrato o sistemas relacionados.

### Solución:
- Asegúrese siempre de que todos los cambios de estado se producen antes de llamar a contratos externos, es decir, actualice los saldos o el código internamente antes de llamar al código externo.
- Utilice modificadores de función que eviten la reentrada, como Re-entrancy Guard de Open Zepplin.

### Ejemplo (Contrato Mejorado):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_Reentrancy {
    mapping(address => uint) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");

        // Corrección: Actualizar el saldo del usuario antes de enviar Ether
        balances[msg.sender] = 0;

        // Luego enviar Ether
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### Ejemplos de Contratos Inteligentes que fueron víctimas de Ataques de Reentrada:
1. [Rari Capital](https://etherscan.io/address/0xe16db319d9da7ce40b666dd2e365a4b8b3c18217#code) : Un Completo [Análisis del Hack](https://blog.solidityscan.com/rari-capital-re-entrancy-vulnerability-analysis-25df2bbfc803)
2. [Orion Protocol](https://etherscan.io/address/0x98a877bb507f19eb43130b688f522a13885cf604#code) : Un Completo [Análisis del Hack](https://blog.solidityscan.com/orion-protocol-hack-analysis-missing-reentrancy-protection-f9af6995acb3)