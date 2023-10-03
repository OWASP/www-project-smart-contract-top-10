### Reentranct Atackes (Ataques de Reentrada)

### Descripción
Un ataque de reentrada ocurre cuando una función es invocada externamente durante su ejecución, permitiendo que sea ejecutada múltiples veces en una única transacción. Esto ocurre típicamente cuando un contrato llama a otro contrato antes de resolver su estado.

### Impacto
Un ataque de reentrada con éxito puede provocar el drenado de fondos, llamadas a funciones no autorizadas o cambios de estado que interrumpan las operaciones normales del contrato.

### Pasos para Solucionarlo
1. Asegúrate de que sigues el patrón Checks-Effects-Interactions (CEI): comprueba las condiciones, luego realiza cambios y después interactúa con otros contratos.
2. Utiliza una guardia de reentrada (guard) o un bloqueo de exclusión mutua (mutex) para bloquear las llamadas recursivas de contratos externos durante la ejecución de una función.
3. Actualice regularmente a la última versión de Solidity, que incluye protección inherente contra ataques de reentrada.

### Ejemplo
El infame pirateo DAO fue un ataque de reentrada. Un atacante explotó una vulnerabilidad de reentrada para drenar alrededor de 3,6 millones de Ether del contrato.