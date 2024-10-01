## API Server
---
##### Funcion:
El API Server es el **punto de entrada central** a travez del cual los usuarios, componentes internos de Kubernetes y  herramientas externas **interactuan** con el cluster. **Es el frontend de Kubernetes**
##### Importancia:
Todos los comandos (ya sea mediante la [[CLI (Kubeclt)]], la interfaz web o API externa) se procestan a travez del API Server.
**==Maneja la autenticacion, autorizacion, validacion y la actualizacion del estado del cluster en etcd.==**
##### Escalabilidad:
Es un componente que puede escalar horizontalmente para manejar grandes volumentes de solicitudes.

**Corre en los Masters  y requiere el puerto 6443**

6443:
- Este es el puerto principal donde el **API Server** escucha las solicitudes entrantes.