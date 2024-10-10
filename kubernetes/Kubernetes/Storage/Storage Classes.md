## Storage Classes

Es un recurso que define diferentes tipos de almacenamiento que se pueden provisionar dinámicamente para los _PersistentVolumeClaims_ (PVCs). 

La _StorageClass_ especifica un "provisioner" (un controlador de almacenamiento) que administra el aprovisionamiento de volúmenes, además de parámetros como el tipo de almacenamiento, las políticas de reaprovisionamiento, y otras configuraciones que se aplican a los volúmenes creados bajo esa clase.

![[storage-class-flow.png]]

Se pueden definir múltiples storage classes especificando el provider de volumen a utilizar cuando se crea un volumen de esa storage class.
Esto permite al administrador del clúster definir varios tipos de almacenamiento dentro de un clúster, cada uno con un conjunto personalizado de parámetros.

Supongamos que en un clúster de Kubernetes se desean ofrecer tres tipos de almacenamiento:

1. **Rápido y Replicado**: Usando SSD y replicación para aplicaciones críticas.
2. **Económico y Escalable**: Usando discos HDD para almacenamiento a gran escala.
3. **NFS Compartido**: Para acceso compartido entre múltiples pods.


Por ejemplo creamos un StorageClass que provisione sobre un NFS:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage-class
provisioner: example.com/nfs
parameters:
  archiveOnDelete: "false"
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

Para utilizar esta _StorageClass_, puedes definir un _PersistentVolumeClaim_ que especifique esta clase de almacenamiento:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-storage-class
  resources:
    requests:
      storage: 10Gi
```

En este ejemplo, el _PersistentVolumeClaim_ (`nfs-pvc`) solicitará 10 GiB de almacenamiento utilizando la clase de almacenamiento `nfs-storage-class`. 
Cuando este PVC es creado, el controlador NFS se encargará de aprovisionar el almacenamiento automáticamente, según los parámetros definidos en la _StorageClass_.