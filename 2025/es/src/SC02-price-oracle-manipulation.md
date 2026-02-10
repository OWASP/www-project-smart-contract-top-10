# SC02:2025 -  Manipulación del precio del oráculo

## Descripción:
Price Oracle Manipulation es una vulnerabilidad crítica en contratos inteligentes que dependen de fuentes de datos externas (oráculos) para obtener precios u otra información. En las finanzas descentralizadas (DeFi), los oráculos se utilizan para proporcionar datos del mundo real, como los precios de los activos, a los contratos inteligentes. Sin embargo, si se manipulan los datos proporcionados por el oráculo, puede dar lugar a un comportamiento incorrecto del contrato. Los atacantes pueden aprovecharse de los oráculos manipulando los datos que proporcionan, lo que puede tener consecuencias devastadoras, como retiradas no autorizadas, apalancamiento excesivo o incluso vaciar las reservas de liquidez. Para evitar este tipo de ataques, es esencial contar con las salvaguardas y los mecanismos de validación adecuados.

## Ejemplo (contrato vulnerable):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0, "Price must be positive");

        // Vulnerabilidad: No hay validación ni protección contra la manipulación de precios
        uint256 collateralValue = uint256(price) * amount;

        // Lógica de préstamo basada en el precio manipulado
        // Si un atacante manipula el oráculo, podría pedir prestado más de lo que debería
    }

    function repay(uint256 amount) public {
        // Lógica de reembolso
    }
}
```

### Impacto:
- Los atacantes podrían manipular el oráculo para inflar el precio de un activo, lo que les permitiría pedir prestados más fondos de los que les corresponderían.
- En los casos en los que el precio manipulado lleve a una evaluacion falsa del colateral/garantia, los usuarios legítimos podrían enfrentarse a la liquidación debido a valoraciones incorrectas.
- Si se compromete un oráculo, los atacantes pueden explotar los datos manipulados para drenar los fondos de liquidez del contrato o incluso provocar la insolvencia de un contrato.

### Solución:
- Agregue datos de múltiples oráculos independientes para reducir el riesgo de manipulación por una sola fuente.
- Establecer umbrales mínimos y máximos para los precios recibidos del oráculo para evitar que las oscilaciones drásticas de precios afecten a la lógica del contrato.
- Introducir un bloqueo temporal entre las actualizaciones de precios para evitar cambios instantáneos que puedan ser aprovechados por los atacantes.
- Utilizar pruebas criptográficas para garantizar la autenticidad de los datos recibidos de los oráculos, como requerir firmas de terceros de confianza.

### Ejemplo (Contrato Mejorado):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IPriceFeed {
    function getLatestPrice() external view returns (int);
}

contract PriceOracleManipulation {
    address public owner;
    IPriceFeed public priceFeed;
    int public minPrice = 1000; // Fijar un precio mínimo aceptable
    int public maxPrice = 2000; // Fijar un precio máximo aceptable

    constructor(address _priceFeed) {
        owner = msg.sender;
        priceFeed = IPriceFeed(_priceFeed);
    }

    function borrow(uint256 amount) public {
        int price = priceFeed.getLatestPrice();
        require(price > 0 && price >= minPrice && price <= maxPrice, "Price manipulation detected");

        uint256 collateralValue = uint256(price) * amount;

        // Lógica de préstamo basada en el precio válido
    }

    function repay(uint256 amount) public {
        // Lógica de reembolso
    }
}
```

### Ejemplos de contratos inteligentes que fueron víctimas de este tipo de ataque :
1. [Análisis del hackeo de Polter Finance](https://blog.solidityscan.com/polter-finance-hack-analysis-c5eaa6dcfd40) 
2. [BonqDAO Protocol](https://polygonscan.com/address/0x4248fd3e2c055a02117eb13de4276170003ca295#code) - [Análisis del Hackeo](https://blog.solidityscan.com/bonqdao-protocol-hack-analysis-oracle-manipulation-8e6978149a66)