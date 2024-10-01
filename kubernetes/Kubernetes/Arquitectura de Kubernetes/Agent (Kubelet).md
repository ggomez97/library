## Agent (Kubelet)
---
#### Funcion:
Es el agente que se ejecuta en cada nodo del cluster, grantiza que los contenedores que pertenecen a los pods se ejecuten y se mantengan en buen estado
#### Interaccion:
Se comunica a travez del [[API Server]] para recibir las especificaciones de los pods y se encarga de iniciar los contenedores, monitorearlos y reiniciarlos si es necesario

**Manejo del estado**: Revisa periódicamente el estado de los contenedores y reporta al `API Server`.

* Vigila los pods que se han asignado a su nodo 
* Monta los volúmenes requeridos del pod 
* Ejecuta los contenedores del pod a través de docker 
* Ejecuta periódicamente sondeos de vida de contenedores 
* Informa del estado del pod al resto del sistema

**Corre en los Masters y requiere los puertos 10250:**

10250:

- l puerto 10250 es el principal puerto utilizado por el `kubelet` para las comunicaciones seguras con otros componentes del clúster, como el **API Server** y los usuarios que ejecutan comandos a través de [[kubectl]].

