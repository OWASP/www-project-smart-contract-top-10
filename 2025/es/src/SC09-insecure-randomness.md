## SC09:2025 - Aleatoriedad Insegura

### Descripción:
Los generadores de números aleatorios son esenciales para aplicaciones como juegos de azar, selección de ganadores y generación de semillas aleatorias. En Ethereum, generar números aleatorios es un desafío debido a su naturaleza determinista. Dado que Solidity no puede producir números verdaderamente aleatorios, depende de factores pseudoaleatorios. Además, los cálculos complejos en Solidity son costosos en términos de gas.

*Mecanismos Inseguros para Crear Números Aleatorios en Solidity: Los desarrolladores suelen utilizar métodos relacionados con bloques para generar números aleatorios, como:*
  - `block.timestamp`: Marca de tiempo del bloque actual.
  - `blockhash(uint blockNumber)`: Hash de un bloque determinado (solo para los últimos 256 bloques).
  - `block.difficulty`: Dificultad del bloque actual.
  - `block.number`: Número del bloque actual.
  - `block.coinbase`: Dirección del minero del bloque actual.
    
Estos métodos son inseguros porque los mineros pueden manipularlos, afectando la lógica del contrato.

### Ejemplo (Contrato Vulnerable):
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract Solidity_InsecureRandomness {
    constructor() payable {}

    function guess(uint256 _guess) public {
        uint256 answer = uint256(
            keccak256(
                abi.encodePacked(block.timestamp, block.difficulty, msg.sender) // Uso de mecanismos inseguros para la generación de números aleatorios
            ) 
        );

        if (_guess == answer) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Fallo al enviar Ether");
        }
    }
}
```

### Impacto:
- La aleatoriedad insegura puede ser explotada por atacantes para obtener una ventaja injusta en juegos, loterías y cualquier otro contrato que dependa de la generación de números aleatorios. Al predecir o manipular los resultados supuestamente aleatorios, los atacantes pueden influir en los resultados a su favor. Esto puede llevar a victorias injustas, pérdidas financieras para otros participantes y una falta general de confianza en la integridad y equidad del contrato inteligente.

### Remediación:
- Uso de oráculos (Oraclize) como fuentes externas de aleatoriedad. Se debe tener cuidado al confiar en el oráculo, y se pueden utilizar múltiples oráculos.
- Uso de esquemas de compromiso — Un primitivo criptográfico que utiliza un enfoque de compromiso-revelación. Tiene amplias aplicaciones en lanzamiento de monedas, pruebas de conocimiento cero y computación segura. Ejemplo: RANDAO.
- Chainlink VRF — Es un generador de números aleatorios (RNG) verificable y demostrablemente justo que permite a los contratos inteligentes acceder a valores aleatorios sin comprometer la seguridad o la usabilidad.
- Algoritmo Signidice — Adecuado para PRNG en aplicaciones que involucran dos partes usando firmas criptográficas.
- Hashes de bloques de Bitcoin — Se pueden utilizar oráculos como BTCRelay, que actúan como un puente entre Ethereum y Bitcoin. Los contratos en Ethereum pueden solicitar hashes de bloques futuros de la blockchain de Bitcoin como fuente de entropía. Cabe señalar que este enfoque no es seguro contra el problema de incentivos de los mineros y debe implementarse con precaución.

### Ejemplo (Versión Corregida):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract Solidity_InsecureRandomness is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    uint256 public randomResult;

    constructor(address _vrfCoordinator, address _linkToken, bytes32 _keyHash, uint256 _fee) 
        VRFConsumerBase(_vrfCoordinator, _linkToken) 
    {
        keyHash = _keyHash;
        fee = _fee;
    }

    function requestRandomNumber() public returns (bytes32 requestId) {
        require(LINK.balanceOf(address(this)) >= fee, "No hay suficiente LINK");
        return requestRandomness(keyHash, fee);
    }

    function fulfillRandomness(bytes32 requestId, uint256 randomness) internal override {
        randomResult = randomness;
    }

    function guess(uint256 _guess) public {
        require(randomResult > 0, "Número aleatorio no generado aún");
        if (_guess == randomResult) {
            (bool sent,) = msg.sender.call{value: 1 ether}("");
            require(sent, "Fallo al enviar Ether");
        }
    }
}
```

### Ejemplos de Contratos Inteligentes que fueron Víctimas de Ataques por Aleatoriedad Insegura:
1. [Hack de Roast Football](https://bscscan.com/address/0x26f1457f067bf26881f311833391b52ca871a4b5#code) : Un Análisis Completo del [Hack](https://blog.solidityscan.com/roast-football-hack-analysis-e9316170c443)
2. [Hack de FFIST](https://bscscan.com/address/0x80121da952a74c06adc1d7f85a237089b57af347#code) : Un Análisis Completo del [Hack](https://blog.solidityscan.com/ffist-hack-analysis-9cb695c0fad9)
