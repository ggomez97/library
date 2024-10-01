## Volume Access Mode
A persistent volume can be mounted on a host in any way supported by the resource provider. Different storage providers have different capabilities and access modes are set to the specific modes supported by that particular volume. For example, NFS can support multiple read write clients, but an iSCSI volume can be support only one.

The access modes are:

  * **ReadWriteOnce**: the volume can be mounted as read-write by a single node
  * **ReadOnlyMany**: the volume can be mounted read-only by many nodes
  * **ReadWriteMany**: the volume can be mounted as read-write by many nodes

Claims and volumes use the same conventions when requesting storage with specific access modes. Pods use claims as volumes. For volumes which support multiple access modes, the user specifies which mode desired when using their claim as a volume in a pod.

A volume can only be mounted using one access mode at a time, even if it supports many. For example, a NFS volume can be mounted as ReadWriteOnce by a single node or ReadOnlyMany by many nodes, but not at the same time.

Block based volumes, e.g. iSCSI and Fibre Channel cannot be mounted as ReadWriteMany at same type. The iSCSI and Fibre Channel volumes do not have any fencing mechanisms yet, so you must ensure the volumes are only used by one node at a time. In certain situations, such as draining a node, the volumes may be used simultaneously by two nodes. Before draining the node, first ensure the pods that use these volumes are deleted.