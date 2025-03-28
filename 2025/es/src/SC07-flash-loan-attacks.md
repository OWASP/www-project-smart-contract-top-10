## SC07:2025 - Ataques de Flash Loans (Préstamos Relámpago) 

### Descripción  
Los ataques de FL (Flash Loans) explotan la capacidad de pedir prestadas grandes sumas de fondos sin necesidad de colateral dentro de una única transacción. Estos ataques aprovechan la naturaleza atómica de las transacciones en blockchain, donde todas las operaciones deben completarse con éxito o revertirse en su totalidad. Al combinar los FL con otras vulnerabilidades como la manipulación de oráculos, la reentrada o lógica defectuosa, los atacantes pueden alterar el comportamiento de los contratos y drenar fondos.  

#### Ejemplos de Explotaciones de FL  
1. **Manipulación de Oráculos:** Uso de fondos prestados para distorsionar oráculos de precios, desencadenando liquidaciones sin suficiente colateral.  
2. **Drenaje de Pools de Liquidez:** Aprovechamiento de FL para retirar liquidez o explotar mecanismos mal diseñados de AMM.  
3. **Explotaciones de Arbitraje:** Manipulación de la liquidez para explotar discrepancias de precios entre plataformas.  

### Impacto  
- **Pérdida de Fondos:** Los atacantes pueden drenar reservas de protocolos o manipular préstamos colateralizados para robar activos.  
- **Disrupción del Mercado:** Manipulación temporal de precios o agotamiento de liquidez que afecta a usuarios y plataformas.  
- **Daño al Ecosistema:** Pérdida de confianza en los protocolos, lo que reduce la adopción por parte de los usuarios y genera impacto financiero.  

### Remediación  
- **Evitar la dependencia de FL en lógica crítica:** Restringir las funciones sensibles para operar solo en condiciones validadas y predecibles.  
- **Diseño Robusto de Oráculos:** Utilizar precios promedio ponderados en el tiempo (TWAP) u oráculos descentralizados resistentes a la manipulación.  
- **Pruebas Exhaustivas:** Incluir pruebas que simulen escenarios de FL y casos extremos.  
- **Control de Acceso:** Limitar el acceso a funciones críticas para evitar transacciones no autorizadas o maliciosas.  

### Ejemplos de Explotaciones de FL  
1. [Hack de UwUlend](https://blog.solidityscan.com/uwulend-hack-analysis-77eb9181a717): Un análisis detallado del [ataque](https://blog.solidityscan.com/uwulend-hack-analysis-77eb9181a717).  
2. [Hack de Doughfina](https://blog.solidityscan.com/doughfina-hack-analysis-685ed56adb19): Un análisis detallado del [ataque](https://blog.solidityscan.com/doughfina-hack-analysis-685ed56adb19).  
