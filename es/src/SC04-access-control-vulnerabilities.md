# Access Control Vulnerabilities (Vulnerabilidades de control de acceso)

### Descripción
Existen vulnerabilidades de control de acceso cuando un contrato no restringe adecuadamente quién puede llamar a ciertas funciones. Esto puede resultar en llamadas a funciones no autorizadas.

### Impacto
Si una función del contrato no está protegida adecuadamente, actores no autorizados pueden manipular el estado del contrato, robar fondos o realizar otras acciones perjudiciales.

### Pasos a seguir
1. Utilice patrones de control de acceso como Ownable o RBAC (Role-Based Access Control) en sus contratos.
2. Audite periódicamente el contrato en busca de posibles vulnerabilidades de control de acceso.
3. Limite las capacidades de las funciones y roles individuales dentro del contrato.

### Ejemplo
La vulnerabilidad de Parity Wallet fue el resultado de una función desprotegida en un contrato de biblioteca, permitiendo a un atacante tomar posesión del contrato y autodestruirlo, congelando más de 500.000 Ether.