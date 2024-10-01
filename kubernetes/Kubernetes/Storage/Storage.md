# Tipos de Storage:
---
- **Persistent Volumes (PV)**: Representan un recurso de almacenamiento real, como un disco físico o una partición en un almacenamiento remoto.
	- NFS
	- iSCSI
	- Almacenamiento en la nube como Amazon EBS, Google Persistent Disk, etc.
  
- **Persistent Volume Claims (PVC)**: Son las solicitudes de los pods para reclamar el espacio de almacenamiento definido en los PV. Los pods no interactúan directamente con los PV, sino a través de los PVC.

Kubertenes tiene 2 formas de provisonar storage:

  * **Provisión Manual**: Útil en entornos donde el almacenamiento es constante y no requiere cambios frecuentes. Por ejemplo, si ya tienes discos físicos o almacenamiento en red configurado y solo deseas reutilizar esos recursos.

  * **Provisión Dinámica**: Ideal para entornos de producción donde necesitas escalar rápidamente y la demanda de almacenamiento puede cambiar con frecuencia. Esto es común en aplicaciones en la nube donde los recursos son efímeros y escalables.


| **Caracterisitca** |                       **Provision Manual**                        |             Provision DInamica             |
| :----------------: | :---------------------------------------------------------------: | :----------------------------------------: |
|  Creacion de PVs   |              Predefinidos por el administrador <br>               |  Generados automáticamente por Kubernetes  |
|    Flexibilidad    |             Menos Flexible, requiere gestion manual.              |    Mas flexible, ajustado a la demanda.    |
|   Configuracion    |                 Necesita especificar el PV exacto                 |   Solo se especifica la `Storage Class`    |
|        Uso         | Util en entornos donde el almacenamiento no cambia frecuentemente | Ideal para entornos dinamicos y escalables |

| Manual                                                    | Dinamica                                                                                           |
| --------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| **PV Manuales**: Creación manual por el administrador.    | **PV Dinámicos**: Creación automática de volúmenes a partir de reclamaciones.                      |
| **PVC Manuales**: Reclamos específicos de PVs existentes. | **PVC Dinámicos**: Reclamos que utilizan `StorageClass` para la provisión automática de volúmenes. |

In this section we're going to introduce this model by using simple examples. Please, refer to official documentation for more details.

  * [[Local Persistent Volumes (LPV)]]
  * [[Volume Access Mode]]
  * [[Volume State]]
  * [[Volume Reclaim Policy]]
  * [[Manual volumes provisioning]]
  * [[Storage Classes]]
  * [[Dynamic volumes provisioning]]
  * [[Redis benchmark]]
  * [[Stateful Applications]]
  * [[Configure GlusterFS as Storage backend]]
