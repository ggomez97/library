## etcd
---
##### Funcion:
	
Es una base de datos clave-valor distribuida y altamente dispnible que almacena todos los datos de configuracion y estado del cluster de Kubernetes.
Es fundamental para la persistencia y recuperacion de informacion sobre el estado del cluster.
##### Importancia:

Guarda el estado de los objetos del cluster como `pods`, `nodos`, `network`, etc.
Cada cambio que ocurre en el cluster (agregar un pod,actualizar un servicio) se registra en `etcd`.
Si se pierde `etcd`, se pierde el estado del cluster.
##### Rendimiento:

Dado que es un componente crítico, debe ser rápido, fiable y estar configurado con redundancia para evitar la pérdida de datos.


**Corre en los Masters  y requiere los puertos 2380 y 2379.**

2380:
- Este puerto es utilizado exclusivamente por los servidores `etcd` (masters) para sincronizar el estado del clúster entre pares (replicación de datos, consenso, etc.).

2379:
- Lo utilizan tanto los componentes que corren en los **nodos de control (masters)** como el propio [[Agent (Kubelet)]] que corre en los **workers** (aunque los workers generalmente no interactúan directamente con `etcd`, lo hacen a través del `API Server`).
