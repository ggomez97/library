## Volume state
When a pod claims for a volume, the cluster inspects the claim to find the volume meeting claim requirements and mounts that volume for the pod. Once a pod has a claim and that claim is bound, the bound volume belongs to the pod.

A volume will be in one of the following state:

  * **Available**: a volume that is not yet bound to a claim
  * **Bound**: the volume is bound to a claim
  * **Released**: the claim has been deleted, but the volume is not yet available
  * **Failed**: the volume has failed 

The volume is considered released when the claim is deleted, but it is not yet available for another claim. Once the volume becomes available again then it can bound to another other claim. 

In our example, delete the volume claim

    kubectl delete pvc volume-claim

See the status of the volume

    kubectl get pv persistent-volume
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Available                                      57m