## SC10:2025 - Denegación de Servicio

### Descripción:

Un ataque de Denegación de Servicio (DoS) en Solidity implica explotar vulnerabilidades para agotar recursos como gas, ciclos de CPU o almacenamiento, haciendo que un contrato inteligente sea inutilizable. Los tipos comunes incluyen ataques de agotamiento de gas, donde actores malintencionados crean transacciones que requieren un consumo excesivo de gas; ataques de reentrada, que explotan secuencias de llamadas de contratos para acceder a fondos no autorizados; y ataques al límite de gas del bloque, que consumen el gas del bloque, obstaculizando las transacciones legítimas.

### Ejemplo (Contrato vulnerable):

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    function claimThrone() external payable {
        require(msg.value > balance, "Debes pagar más para convertirte en el rey");

        // Si el actual rey tiene una función fallback maliciosa que revierte, evitará que un nuevo rey reclame el trono, causando una Denegación de Servicio.
        (bool sent,) = king.call{value: balance}("");
        require(sent, "No se pudo enviar Ether");

        balance = msg.value;
        king = msg.sender;
    }
}
```

### Impacto:

- Un ataque DoS exitoso puede hacer que el contrato inteligente quede inoperable, impidiendo que los usuarios interactúen con él como se espera. Esto puede interrumpir operaciones y servicios críticos que dependen del contrato.
- Los ataques DoS pueden provocar pérdidas financieras, especialmente en aplicaciones descentralizadas (dApps) donde los contratos inteligentes gestionan fondos o activos.
- Un ataque DoS puede dañar la reputación del contrato inteligente y su plataforma asociada. Los usuarios pueden perder confianza en la seguridad y fiabilidad de la plataforma, lo que podría derivar en la pérdida de usuarios y oportunidades de negocio.

### Remediación:

- Asegurar que los contratos inteligentes puedan manejar fallos constantes, como el procesamiento asincrónico de llamadas externas que podrían fallar, para mantener la integridad del contrato y evitar comportamientos inesperados.
- Ser cauteloso al usar `call` para llamadas externas, bucles y recorridos para evitar un consumo excesivo de gas, lo que podría provocar fallos en las transacciones o costos inesperados.
- Evitar otorgar demasiados permisos a un solo rol en los contratos. En su lugar, dividir los permisos de manera razonable y usar administración con billeteras multifirma para roles con permisos críticos, previniendo la pérdida de acceso en caso de compromiso de una clave privada.

### Ejemplo (Versión corregida):

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.24;

contract Solidity_DOS {
    address public king;
    uint256 public balance;

    // Usar un método más seguro para transferir fondos, como transfer, que tiene una asignación fija de gas.
    // Esto evita el uso de call y previene problemas con funciones fallback maliciosas.
    function claimThrone() external payable {
        require(msg.value > balance, "Debes pagar más para convertirte en el rey");

        address previousKing = king;
        uint256 previousBalance = balance;

        // Actualizar el estado antes de transferir Ether para prevenir problemas de reentrada.
        king = msg.sender;
        balance = msg.value;

        // Usar transfer en lugar de call para asegurar que la transacción no falle debido a una función fallback maliciosa.
        payable(previousKing).transfer(previousBalance);
    }
}
```

