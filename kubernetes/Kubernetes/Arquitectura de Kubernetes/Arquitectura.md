# Arquitectura del Sistema
---

Kubernetes es una plataforma de código abierto para automatizar la implementación, el escalado y las operaciones de contenedores de aplicaciones en un clúster de hosts. Con Kubernetes, puede responder de manera rápida y eficiente a la demanda de los clientes:

 * Implemente sus aplicaciones de manera rápida y predecible.
 * Escale sus aplicaciones sobre la marcha. 
 * Implementación sin problemas de nuevas funciones. 
 * Optimice el uso de su hardware utilizando solo los recursos que necesita
   
 Un sistema Kubernetes está formado por muchos nodos (tanto hosts físicos como virtuales) que forman un clúster. 
 Un nodo del clúster tiene un rol, hay nodos **Masters y Workers**. Los **Masters** ejecutan el **Control Plane** y los **Workers** ejecutan las aplicaciones de usuario. Se utilizan varios **Masters** para la alta disponibilidad del plano de control.
 
Diagrama de arquitectura de Kubernetes:

![[architecture.jpg]]

Componentes centrales de un **Cluster de Kubernetes**:

   * [[etcd]]
   * [[API Server]]
   * [[Controller Manager]]
   * [[Scheduler]]
   * [[Agent (Kubelet)]]
   * [[Proxy]]
   * [[CLI (Kubeclt)]]
  












