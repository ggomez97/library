## Manual volumes provisioning
In this section we're going to use a **Network File System** storage backend for manual provisioning of shared volumes. Main limit of local storage for container volumes is that storage area is tied to the host where it resides. If kubernetes moves the pod from another host, the moved pod is no more to access the data since local storage is not shared between multiple hosts of the cluster. To achieve a more useful storage backend we need to leverage on a shared storage technology like NFS.

We'll assume a simple external NFS server ``fileserver`` sharing some folders. To make worker nodes able to consume these NFS shares, install the NFS client on all the worker nodes by ``yum install -y nfs-utils`` command.

Define a persistent volume as in the ``nfs-persistent-volume.yaml`` configuration file
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-volume
spec:
  storageClassName: ""
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    path: "/mnt/nfs"
    server: fileserver
  persistentVolumeReclaimPolicy: Recycle
```

Create the persistent volume

    kubectl create -f nfs-persistent-volume.yaml
    persistentvolume "nfs" created

    kubectl get pv nfs -o wide
    NAME        CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    nfs-volume  1Gi        RWO           Recycle         Available                                      7s

Thanks to the persistent volume model, kubernetes hides the nature of storage and its complex setup to the applications. An user need only to claim volumes for their pods without deal with storage configuration and operations.

Create the claim

    kubectl create -f volume-claim.yaml

Check the bound 

    kubectl get pv
    NAME        CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
    nfs-volume  1Gi        RWO           Recycle         Bound     project/volume-claim                            5m

    kubectl get pvc
    NAME           STATUS    VOLUME      CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    volume-claim   Bound     nfs-volume  1Gi        RWO                          9s

Now we are going to create more nginx pods using the same claim.

For example, create the ``nginx-pvc-template.yaml`` template for a nginx application having the html content folder placed on the shared storage 
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  generation: 1
  labels:
    run: nginx
  name: nginx-pvc
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:latest
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
          name: "http-server"
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: html
      volumes:
      - name: html
        persistentVolumeClaim:
          claimName: volume-claim
      dnsPolicy: ClusterFirst
      restartPolicy: Always
```

The template above defines a nginx application based on a nginx deploy of 3 replicas. The nginx application requires a shared volume for its html content. The application does not have to deal with complexity of setup and admin an NFS share.

Deploy the application

    kubectl create -f nginx-pvc-template.yaml
    
Check all pods are up and running

    kubectl get pods -o wide
    NAME                         READY     STATUS    RESTARTS   AGE       IP            NODE
    nginx-pvc-3474572923-3cxnf   1/1       Running   0          2m        10.38.5.89    kubew05
    nginx-pvc-3474572923-6cr28   1/1       Running   0          6s        10.38.3.140   kubew03
    nginx-pvc-3474572923-z17ls   1/1       Running   0          2m        10.38.5.90    kubew05

Login to one of these pods and create some html content

    kubectl exec -it nginx-pvc-3474572923-3cxnf bash
    root@nginx-pvc-3474572923-3cxnf:/# echo 'Hello from NFS!' > /usr/share/nginx/html/index.html                
    root@nginx-pvc-3474572923-3cxnf:/# exit

Since all three pods mount the same shared folder on the NFS, the just created html content is placed on the NFS share and it is accessible from any of the three pods

    curl 10.38.5.89    
    Hello from NFS!
    
    curl 10.38.5.90
    Hello from NFS!
    
    curl 10.38.3.140
    Hello from NFS!

### Volume selectors
A volume claim can define a label selector to bound a specific volume. For example, define a claim as in the ``pvc-volume-selector.yaml`` configuration file
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-volume-selector
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      volumeName: "share01"
```

Create the claim

	kubectl create -f pvc-volume-selector.yaml

The claim remains pending because there are no matching volumes

	kubectl get pvc
	NAME                  STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
	pvc-volume-selector   Pending                                                      5s
	
	kubectl get pv
	NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     
	share01   1Gi        RWX            Recycle          Available             
	share02   1Gi        RWX            Recycle          Available             

Pick the volume named ``share01`` and label it

	kubectl label pv share00 volumeName="share01"
	persistentvolume "share01" labeled
	
And check if the claim bound the volume

	kubectl get pv
	NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                         
	share01   1Gi        RWX            Recycle          Bound       project/pvc-volume-selector     
	share02   1Gi        RWX            Recycle          Available                                 
