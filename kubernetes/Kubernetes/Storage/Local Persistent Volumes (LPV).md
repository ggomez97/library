Se empieza definiendo un persistent volume (volumen persistente).

 ``local-persistent-volume-recycle.yaml`` 
```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: volumen-local
  labels:
    type: volumen-local
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

En el yml anteriror se especifica que el tipo de volumen es  `hostPath`, de esta forma el mismo va a estar de forma local en el nodo host en `/data` con un espacio disponible de `2Gb`.

El acceso al mismo es de `ReadWriteOnce` lo que implica que solo puede ser montado con el permiso `write` en un solo `pod` al a vez, con `Recicle` logramos que el volumen se reutilice N veces.
 The reclaim policy is ``Recycle`` meaning the volume can be used many times.  It defines the Storage Class name manual for the persisten volume, which will be used to bind a claim to this volume.

Crea el VP (Volumen Persistente)

    kubectl create -f local-persistent-volume-recycle.yaml
    
Muestra informacion del mismo:

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Available                                      33m

Luego de crear el volumen persistente debemos hacer un `claim` al mismo  para persistir nuestro datos.

``volume-claim.yaml`` 
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
En la configuracion esta especificado 1GB pero el volumen que creamos previamente es de 2GB, lo que pasa es que el `claim` va hacer un `bound` a cualquier PV que tenga los requisitos minimos definidos.

Creacion del claim

    kubectl create -f volume-claim.yaml

Status  del volumen persistente para ver  si esta en estado `bound`

    kubectl get pv
    NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
    local     2Gi        RWO           Recycle         Bound     project/volume-claim                            37m

Status del `claim`

    kubectl get pvc
    NAME           STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS   AGE
    volume-claim   Bound     local     2Gi        RWO                          1m

Creamos un  pod de nginx

``nginx-pod-pvc.yaml`` 
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

Cuando queremos usar un PV al nodo le tenemos que indicar el PVC, ya que los pods se comunican a travez de los PVC.

**Los PVC deben estar en el mismo** `namespace`

Creamos el pod de nginx

    kubectl create -f nginx-pod-pvc.yaml

Vemos la IP del POD y lo testeamos con `curl`

    kubectl get pod nginx -o yaml | grep IP
      hostIP: 10.10.10.86
      podIP: 172.30.5.2

    curl 172.30.5.2:80
    403 Forbidden

En el nodo creamos un archivo `index.html`dentro de `/data` con un texto.

    echo "Welcome to $(hostname)" > /data/index.html

Resultado del nuevo `curl`

     curl 172.30.5.2:80
     Welcome to kubew05

Podemos probar la persistencia del volumen borrando el pod para que se recree, si nada falla  nuestra web `index.html` deberia estar disponible.

    kubectl delete pod nginx
    pod "nginx" deleted

    kubectl create -f nginx-pod-pvc.yaml
    pod "nginx" created

    kubectl get pod nginx -o yaml | grep podIP
      podIP: 172.30.5.2

    curl 172.30.5.2
    Welcome to kubew05
