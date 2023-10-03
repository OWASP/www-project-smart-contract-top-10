# Integer Overflow and Underflow (Desbordamiento y subdesbordamiento de enteros)

### Descripción
El desbordamiento y el subdesbordamiento de enteros se producen cuando las operaciones aritméticas superan el tamaño máximo o mínimo que puede contener una variable de tipo entero, provocando que el valor se desborde hacia el extremo opuesto.

### Impacto
Un atacante puede explotar estas vulnerabilidades para alterar la lógica del contrato, posiblemente robando activos o acuñando una cantidad excesiva de tokens.

### Pasos a seguir
1. Utilizar SafeMath o librerías similares que ofrezcan funciones para operaciones aritméticas seguras.
2. Actualice a Solidity versión 0.8.0 o posterior, que incluye protección integrada contra desbordamiento y subdesbordamiento.

### Ejemplo
El exploit BatchOverflow en múltiples contratos inteligentes ERC20 permitía a los atacantes generar una cantidad casi infinita de tokens explotando una vulnerabilidad de desbordamiento de enteros.