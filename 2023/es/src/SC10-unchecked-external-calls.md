# Unchecked External Calls (Llamadas externas no comprobadas)

### Descripción
En Ethereum, cuando un contrato llama a otro contrato, el contrato llamado puede fallar silenciosamente sin lanzar una excepción. Si el contrato que llama no comprueba el resultado de la llamada, podría asumir que la llamada fue exitosa, incluso si no lo fue.

### Impacto
Las llamadas externas no comprobadas pueden provocar transacciones fallidas, pérdida de fondos o un estado incorrecto del contrato.

### Pasos para solucionarlo
1. Compruebe siempre el valor de retorno de `call`, `delegatecall` y `callcode`.
2. Utilice las funciones `transfer` o `send` de Solidity en lugar de `call.value()()`, ya que éstas revierten automáticamente en caso de fallo.

### Ejemplo
Un contrato utiliza la función `call` para enviar Ether a una dirección. Si la llamada falla (por ejemplo, si el destinatario es un contrato sin una función fallback pagable), el contrato remitente podría asumir incorrectamente que la transferencia se ha realizado con éxito.