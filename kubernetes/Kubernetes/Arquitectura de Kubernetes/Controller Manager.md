## Controller Manager
---

##### Funcion:
Es el que se encarga de ejecutar varios controladores que observan el estado del cluster mediante el [[API Server]] y se aseguran de que el estado actual coincide con el estado deseado.

**Tipos de controladores:**
- **Node Controller:** Monitorea los nodos si **fallan** y **reacciona** frente a situaciones.
- **Replication Controller:** Monitorea el que un numero especifico de `pods` se esten ejecutando.
- **Endpoint Controller:** Maneja la asociacion de `servicios` y `pods`.
- **Service Account Controller:** Crea cuentas de servicio de tokens y API predeterminados para los nuevos namespaces.

**Ciclo de Retroalimentacion:** Si detecta que el estado deseado no coincide con el actual, actua para corregirlo **(ej: si falla un pod, intenta recrearlo)**

**Corre en los Masters  y requiere el puerto 10252.**

10252:
- Se utiliza para habilitar un **endpoint de métricas no autenticado** (por defecto) al que se puede acceder para obtener información sobre el estado del `Controller Manager`. Las métricas y endpoints de salud en este puerto suelen ser monitoreados por herramientas de observabilidad como **Prometheus** o por sistemas de monitoreo internos de Kubernetes.
