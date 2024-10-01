## Storage Classes
A Persistent Volume uses a given storage class specified into its definition file. A claim can request a particular class by specifying the name of a storage class in its definition file. Only volumes of the requested class can be bound to the claim requesting that class.

If the storage class is not specified in the persistent volume definition, the volume has no class and can only be bound to claims that do not require any class. 

Multiple storage classes can be defined specifying the volume provisioner to use when creating a volume of that class. This allows the cluster administrator to define multiple type of storage within a cluster, each with a custom set of parameters.

For example, the following ``gluster-storage-class.yaml`` configuration file defines a storage class for a GlusterFS backend
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: glusterfs
  labels:
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
parameters:
  resturl: "http://heketi:8080"
  volumetype: "replicate:3"
```

Create the storage class

    kubectl create -f gluster-storage-class.yaml
    kubectl get sc
    NAME                              PROVISIONER
    glusterfs-storage-class           kubernetes.io/glusterfs


The cluster administrator can define a class as default storage class by setting an annotation in the class definition file
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: default-storage-class
  labels:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
parameters:
  resturl: "http://heketi:8080"
```

Check the storage classes

    kubectl get sc
    NAME                              PROVISIONER
    default-storage-class (default)   kubernetes.io/glusterfs
    glusterfs-storage-class           kubernetes.io/glusterfs

If the cluster administrator defines a default storage class, all claims that do not require any class will be dynamically bound to volumes having the default storage class. 