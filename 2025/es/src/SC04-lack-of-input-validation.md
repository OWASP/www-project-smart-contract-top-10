# SC04:2025 - Falta de validación de entrada

## Descripción:
La validación de entrada asegura que un contrato inteligente procesa sólo datos válidos y esperados. Cuando los contratos no validan las entradas entrantes, se exponen inadvertidamente a riesgos de seguridad como la manipulación lógica, el acceso no autorizado y el comportamiento inesperado.Por ejemplo, si un contrato asume que las entradas del usuario son siempre válidas sin verificación, los atacantes pueden explotar esta confianza para introducir datos maliciosos. Esta falta de validación de las entradas compromete la seguridad y la fiabilidad del contrato inteligente.

## Ejemplo (Contrato Vulnerable):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_LackOfInputValidation {
    mapping(address => uint256) public balances;

    function setBalance(address user, uint256 amount) public {
        // La función permite a cualquiera establecer saldos arbitrarios para cualquier usuario sin validación.
        balances[user] = amount;
    }
}
```
### Impacto:
- Los atacantes pueden manipular las entradas para drenar fondos, robar tokens o causar otros daños financieros.
- Entradas inapropiadas pueden corromper variables de estado, llevando a un comportamiento del contrato poco fiable e inseguro.
- Los atacantes pueden explotar el contrato para realizar transacciones u operaciones no autorizadas, afectando tanto al usuario como al sistema en general.

### Remediación:
- Asegúrese de que las entradas se ajustan al tipo esperado.
- Validar que las entradas se encuentran dentro de los límites aceptables.
- Asegurarse de que sólo las entidades autorizadas pueden invocar funciones específicas.
- Valide la estructura de las entradas, como los formatos de dirección o la longitud de las cadenas.
- Detener siempre la ejecución y proporcionar mensajes de error claros cuando las entradas no superen la validación.

### Ejemplo (Contrato Mejorado):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract LackOfInputValidation {
    mapping(address => uint256) public balances;
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Caller is not authorized");
        _;
    }

    function setBalance(address user, uint256 amount) public onlyOwner {
        require(user != address(0), "Invalid address");
        balances[user] = amount;
    }
}
```
### Ejemplos de Contratos Inteligentes que fueron víctimas de ataques debido a la Falta de Validación de Entrada:
1. [Convergence Finance](https://etherscan.io/address/0x2b083beaaC310CC5E190B1d2507038CcB03E7606#code) : Un exhaustivo [Análisis de Hack](https://blog.solidityscan.com/convergence-finance-hack-analysis-12e6acd9ea08)
2. [Socket Gateway](https://etherscan.io/address/0x3a23F943181408EAC424116Af7b7790c94Cb97a5#code) : Un exhaustivo [Análisis Hack](https://blog.solidityscan.com/socket-gateway-hack-analysis-b0e9567f7d3e)