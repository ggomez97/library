## Scheduler
---
#### Funcion:

Se encarga de asignar nuevos pods a nodos del cluster.
Toma decisiones sobre que nodo es el mas adecuado para ejecutar un pod basandose enfactores como la carga de nodos, recursos disponiblesm afinidad, tolerancias, etc.

Cuando se crea un nuevo pod que no tiene nodo asignado el `Scheduler` determina el mejor nodo para ejecutarlo utilizando politicas de asignacion y restricciones que el usaurio pueda haber definido.


**Corre en los Masters  y requiere el puerto 10251**

10251:
- El **Scheduler** de Kubernetes utiliza el **puerto 10251** para exponer un **endpoint HTTP** que proporciona métricas y un chequeo de salud de este componente.