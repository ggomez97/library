The request is not authorized because credentials are missing or invalid.## Local Persistent Volumes

**LocThe request is not authorized because credentials are missing or invalid.
Start by defining a persistent volume ``local-persistent-volume-recycle.yaml`` configuration file.
```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: local
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
  persistentVolumeReclaimPolicy: Recycle
```

The configuration file specifies that the volume is at ``/data`` on the the cluster’s node. The volume type is ``hostPath`` meaning the volume is local to the host node. The configuration also specifies a size of 2GB and the access mode of ``ReadWriteOnce``, meanings the volume can be mounted as read write by a single pod at time. The reclaim policy is ``Recycle`` meaning the volume can be used many times.  It defines the Storage Class name manual for the persisten volume, which will be used to bind a claim to this volume.

Create the persistent volume

    kubectl create -f local-persistent-volume-recycle.yaml
    
and view information about it 

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Available                                      33m

Now, we're going to use the volume above by creating a claiming for persistent storage. Create the following ``volume-claim.yaml`` configuration file
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: volume-claim
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Note the claim is for 1GB of space where the the volume is 2GB. The claim will bound any volume meeting the minimum requirements specified into the claim definition. 

Create the claim

    kubectl create -f volume-claim.yaml

Check the status of persistent volume to see if it is bound

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Bound     project/volume-claim                            37m

Check the status of the claim

    kubectl get pvc
    NAME           STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    volume-claim   Bound     local     2Gi        RWO                          1m

Create a ``nginx-pod-pvc.yaml`` configuration file for a nginx pod using the above claim for its html content directory
```yaml
---
kind: Pod
apiVersion: v1
metadata:
  name: nginx
  namespace: default
  labels:
spec:

  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
      - mountPath: "/usr/share/nginx/html"
        name: html

  volumes:
    - name: html
      persistentVolumeClaim:
       claimName: volume-claim
```

Note that the pod configuration file specifies a persistent volume claim, but it does not specify a persistent volume. From the pod point of view, the claim is the volume. Please note that a claim must exist in the same namespace as the pod using the claim.

Create the nginx pod

    kubectl create -f nginx-pod-pvc.yaml

Accessing the nginx will return *403 Forbidden* since there are no html files to serve in the data volume

    kubectl get pod nginx -o yaml | grep IP
      hostIP: 10.10.10.86
      podIP: 172.30.5.2

    curl 172.30.5.2:80
    403 Forbidden

Let's login to the worker node and populate the data volume

    echo "Welcome to $(hostname)" > /data/index.html

Now try again to access the nginx application

     curl 172.30.5.2:80
     Welcome to kubew05

To test the persistence of the volume and related claim, delete the pod and recreate it

    kubectl delete pod nginx
    pod "nginx" deleted

    kubectl create -f nginx-pod-pvc.yaml
    pod "nginx" created

Locate the IP of the new nginx pod and try to access it

    kubectl get pod nginx -o yaml | grep podIP
      podIP: 172.30.5.2

    curl 172.30.5.2
    Welcome to kubew05
