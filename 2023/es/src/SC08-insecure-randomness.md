### Insecure Randomness (Aleatoriedad Insegura)

### Descripción
Generar aleatoriedad en Ethereum es un reto porque cada nodo debe llegar a la misma conclusión sobre el estado del blockchain. Por lo tanto, los enfoques ingenuos para generar aleatoriedad pueden ser manipulados por mineros o atacantes observadores.

### Impacto
La aleatoriedad insegura puede ser explotada por atacantes para obtener una ventaja injusta en juegos, loterías o cualquier otro contrato que dependa de la generación de números aleatorios.

### Pasos para solucionarlo
1. Utilizar esquemas commit-reveal, en los que los usuarios envían valores hash y los revelan más tarde, para generar aleatoriedad.
2. Utilizar servicios de oráculo externos que proporcionen números aleatorios.

### Ejemplo
Un contrato inteligente de lotería que utilice `block.timestamp` para generar un número aleatorio puede ser manipulado por un minero, haciendo que la lotería sea injusta.