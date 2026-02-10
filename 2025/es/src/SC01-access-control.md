
## SC01:2025 - Control de acceso inadecuado 

### Descripción:
Una vulnerabilidad de control de acceso es un fallo de seguridad que permite a usuarios no autorizados acceder o modificar los datos o funciones del contrato. Estas vulnerabilidades surgen cuando el código del contrato no restringe adecuadamente el acceso en función de los niveles de permiso de los usuarios. El control de acceso en los contratos inteligentes puede estar relacionado con la gobernanza y la lógica crítica, como la emisión de tokens, la votación de propuestas, la retirada de fondos, la pausa y actualización de los contratos y el cambio de titularidad.

### Ejemplo (Contrato vulnerable):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Solidity_AccessControl {
    mapping(address => uint256) public balances;

    // Función Burn sin control de acceso
    function burn(address account, uint256 amount) public {
        _burn(account, amount);
    }
}
```
### Impacto:
- Los atacantes pueden obtener acceso no autorizado a funciones y datos críticos dentro del contrato, comprometiendo su integridad y seguridad.
- Las vulnerabilidades pueden conducir al robo de fondos o activos controlados por el contrato, causando importantes daños financieros a los usuarios y partes interesadas.

### Solución:
- Asegúrese de que las funciones de inicialización sólo pueden ser llamadas una vez y exclusivamente por entidades autorizadas.
- Utilice patrones de control de acceso establecidos como Ownable o RBAC (Role-Based Access Control) en sus contratos para gestionar los permisos y garantizar que sólo los usuarios autorizados puedan acceder a determinadas funciones. Esto puede hacerse añadiendo modificadores de control de acceso apropiados, como `onlyOwner` o roles personalizados a funciones sensibles.

### Ejemplo (Versión Corregida):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// Importar el contrato Ownable de OpenZeppelin para gestionar la propiedad
import "@openzeppelin/contracts/access/Ownable.sol";

contract Solidity_AccessControl is Ownable {
    mapping(address => uint256) public balances;

    // Función Burn con control de acceso adecuado, sólo accesible por el propietario del contrato
    function burn(address account, uint256 amount) public onlyOwner {
        _burn(account, amount);
    }
}
```

### Ejemplos de Contratos Inteligentes Víctimas de Ataques de Control de Acceso Inadecuado:
1. [HospoWise Hack](https://etherscan.io/address/0x952aa09109e3ce1a66d41dc806d9024a91dd5684#code) : Un exhaustivo [Análisis del Hackeo](https://blog.solidityscan.com/access-control-vulnerabilities-in-smart-contracts-a31757f5d707)
2. [LAND NFT Hack](https://bscscan.com/address/0x1a62fe088F46561bE92BB5F6e83266289b94C154#code) : Un completo [Análisis del Hackeo](https://blog.solidityscan.com/land-hack-analysis-missing-access-control-66fb9555a3e3)