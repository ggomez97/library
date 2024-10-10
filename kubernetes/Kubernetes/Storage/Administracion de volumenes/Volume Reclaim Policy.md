## Volume Reclaim Policy

Hay 2 tipos de politicas:
### **Retain**:
- El volumen no se elimina automáticamente cuando el PVC es liberado.
- El volumen se mantiene disponible y requiere intervención manual del administrador para su limpieza o reutilización.
- Útil si deseas conservar los datos para su posterior análisis o migración.

`PV-retain.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
```

### **Delete**:

- El volumen y su contenido son eliminados automáticamente cuando el PVC es liberado.
- Comúnmente usado con volúmenes dinámicos, donde los datos no necesitan persistir una vez que el PVC ha sido liberado.
- Se utiliza frecuentemente con proveedores de almacenamiento en la nube (por ejemplo, EBS en AWS, Persistent Disks en GCP).



`PV-delete.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
```

---

### **Recycle** (deprecado):

- Esta política intentaba limpiar el contenido del volumen (borrando los archivos), dejándolo listo para ser reutilizado.
- Fue deprecada en Kubernetes 1.11 debido a problemas de seguridad y ya no es compatible en versiones más recientes.
- Se recomienda usar **Retain** o **Delete** en lugar de **Recycle**.

---

### **Recomendaciones**:

- **Retain**: Si necesitas preservar los datos después de que un PVC los libera y gestionarlos manualmente.
- **Delete**: Si prefieres que el volumen sea eliminado automáticamente cuando ya no se necesita.
- **Recycle**: No recomendado en versiones recientes; debes evitar su uso.




When deleting a claim, the volume becomes available to other claims only when the volume claim policy is set to ``Recycle``. Volume claim policies currently supported are:

  * **Retain**: the content of the volume still exists when the volume is unbound and the volume is released
  * ~~**Recycle**: the content of the volume is deleted when the volume is unbound and the volume is available #~~ DEPRECADA EN 1.11
  * **Delete**: the content and the volume are deleted when the volume is unbound. 
  
*Please note that, currently, only NFS and HostPath support recycling.* 

When the policy is set to ``Retain`` the volume is released but it is not yet available for another claim because the previous claimant’s data are still on the volume.

Define a persistent volume ``local-persistent-volume-retain.yaml`` configuration file

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local-retain
  labels:
    type: local
spec:
  storageClassName: ""
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data"
  persistentVolumeReclaimPolicy: Retain
```

Create the persistent volume and the claim

    kubectl create -f local-persistent-volume-retain.yaml
    kubectl create -f volume-claim.yaml

Login to the pod using the claim and create some data on the volume

    kubectl exec -it nginx bash
    root@nginx:/# echo "Hello World" > /usr/share/nginx/html/index.html
    root@nginx:/# exit

Delete the claim

    kubectl delete pvc volume-claim

and check the status of the volume

    kubectl get pv
    NAME           CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS     CLAIM                  STORAGECLASS     AGE
    local-retain   2Gi        RWO           Retain          Released   project/volume-claim                    3m
    
We see the volume remain in the released status and not becomes available since the reclaim policy is set to ``Retain``. Now login to the worker node and check data are still there.

An administrator can manually reclaim the volume by deleteting the volume and creating a another one.