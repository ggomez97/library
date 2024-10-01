## Volume Reclaim Policy
When deleting a claim, the volume becomes available to other claims only when the volume claim policy is set to ``Recycle``. Volume claim policies currently supported are:

  * **Retain**: the content of the volume still exists when the volume is unbound and the volume is released
  * **Recycle**: the content of the volume is deleted when the volume is unbound and the volume is available
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