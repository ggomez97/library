## Dynamic volumes provisioning
In this section we're going to use a **GlusterFS** distributed storage backend for dynamic provisioning of shared volumes. We'll assume an external GlusterFS cluster made of three nodes providing a distributed and high available file system.

### Dynamically Provision a GlusterFS volume
Define a storage class for the gluster provisioner
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: glusterfs-storage-class
  labels:
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
parameters:
  resturl: "http://heketi:8080"
  volumetype: "replicate:3"
```

Make sure the ``resturl`` parameter is reporting the Heketi server and port.

Create the storage class

	kubectl create -f gluster-storage-class.yaml

To make the kubernetes worker nodes able to consume GlusterFS volumes, install the gluster client on all worker nodes

	yum install -y glusterfs-fuse

Define a volume claim in the gluster storage class

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: apache-volume-claim
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
  storageClassName: glusterfs-storage-class
```

and an apache pod that is using that volume claim for its static html repository

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apache-gluster-pod
  labels:
    name: apache-gluster-pod
spec:
  containers:
  - name: apache-gluster-pod
    image: centos/httpd:latest
    ports:
    - name: web
      containerPort: 80
      protocol: TCP
    volumeMounts:
    - mountPath: "/var/www/html"
      name: html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: apache-volume-claim
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

Create the storage class, the volume claim and the apache pod

	kubectl create -f glusterfs-storage-class.yaml
	kubectl create -f pvc-gluster.yaml
	kubectl create -f apache-pod-pvc.yaml

Check the volume claim

	kubectl get pvc
	NAME                  STATUS    VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS              AGE
	apache-volume-claim   Bound     pvc-4af76e0f   1Gi        RWX            glusterfs-storage-class   7m

The volume claim is bound to a dynamically created volume on the gluster storage backend

	kubectl get pv
	NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                 STORAGECLASS
	pvc-4af76e0f   1Gi        RWX            Delete           Bound     apache-volume-claim   glusterfs-storage-class

Cross check through the Heketi

 	heketi-cli --server http://heketi:8080 volume list
	Id:7ce4d0cbc77fe36b84ca26a5e4172dbe Name:vol_7ce4d0cbc77fe36b84ca26a5e4172dbe ...

In the same way, the gluster volume is dynamically removed when the claim is removed

	kubectl delete pvc apache-volume-claim
	kubectl get pvc,pv
	No resources found.