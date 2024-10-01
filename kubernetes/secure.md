# Secure Access
In this section we are going to cover additional concepts related to the authentication, authorization and user management.

   * [Service Accounts](#service-accounts)
   * [Authentication](#authentication)
   * [Authorization](#authorization)
   * [Admission Control](#admission-control)
   * [Accessing the APIs server](#accessing-the-apis-server)
   

In kubernetes, users access the API server via HTTP(S) requests. When a request reaches the API, it goes through several stages:

  1. **Authentication**: who can access?
  2. **Authorization**: what can be accessed?
  3. **Admission Control**: cluster wide policy

## Service Accounts
In kubernetes we have two kind of users:

  1. **Service Accounts**: used for applications running in pods 
  2. **User Accounts**: used for humans or machines 
  
User accounts are global, i.e. user names must be unique across all namespaces of a cluster while service accounts are namespaced. Each namespace has a default service account automatically created by kubernetes. 

Access the master node and query for the service accounts

    kubectl get sa --all-namespaces
    NAMESPACE     NAME                 SECRETS   AGE
    default       default              1         3d
    kube-public   default              1         3d
    kube-system   default              1         3d
    project       default              1         3d

Each pod running in the cluster is forced to use the default service account if no one is specified in the pod configuration file. This job is ensured by the ``--admission-control=ServiceAccount`` set in the API server. For example, create an nginx pod and inspect it

    kubectl create -f nginx-pod.yaml
    kubectl get pod nginx -o yaml
    
We can see the default service account assigned to the pod and mounted as a secret volume containing the token for authenticate the pod against the API server
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: project
spec:
  containers:
  - image: nginx:latest
    name: mynginx
...
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-xr7kb
      readOnly: true
...
  serviceAccount: default
  serviceAccountName: default
...
  volumes:
  - name: default-token-xr7kb
    secret:
      defaultMode: 420
      secretName: default-token-xr7kb
```

If we want to use custom service account we have to create it as in the ``nginx-sa.yaml`` configuration file
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
  namespace:
```

and use it in the ``nginx-pod-sa.yaml`` configuration file
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod-sa
  namespace:
  labels:
spec:
  containers:
  - name: mynginx
    image: nginx:latest
    ports:
    - containerPort: 80
  serviceAccount: nginx
```

Inspecting the service account just created above
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
  namespace: project
  selfLink: /api/v1/namespaces/project/serviceaccounts/nginx
  uid:
secrets:
- name: nginx-token-9dx47
```

we see that a service account token has automatically been created 

    kubectl get secrets
    
    NAME                  TYPE                                  DATA      AGE
    default-token-xr7kb   kubernetes.io/service-account-token   3         3d
    nginx-token-9dx47     kubernetes.io/service-account-token   3         22m

Service accounts are tied to a set of credentials stored as secrets which are mounted into pods allowing them to authenticate against the API server.

To achieve token creation for service accounts, we have to pass a private key file to the controller manager via the ``--service-account-private-key-file`` option. The controller manager will use the private key to sign the generated service account tokens. Similarly, we have to pass the corresponding public key to the API server using the ``--service-account-key-file`` option.

*Please, note that each time the public and private certificate keys change, we have to delete the service accounts, including the default service account for each namespace.*

## Authentication
Kubernetes uses different ways to authenticate users: certificates, tokens, passwords as long enhanced methods as OAuth. Multiple methods can be used at same time, depending on the use case. At least two methods:

  * tokens for service accounts
  * at least one of the following methods for user accounts: certificates, tokens, passwords, other
  
When multiple authenticator modules are enabled, the first module to successfully authenticate the request applies since the API server does not guarantee the order authenticators. See the official documentation for all details.

### Certificates
Client certificate authentication is enabled by passing the ``--client-ca-file`` option to API server. The file contains one or more certificates authorities to use to validate client certificates presented to the API server. When a client certificate is presented and verified, the common name field in the subject certificate is used as the user name for the request and the organization fields in the certificate can indicate the group memberships. To include multiple group memberships for a user, include multiple organization fields in the certificate.

For example, to create a certificate for an ``admin`` user belonging the the ``system:masters`` group create a signing request ``admin-csr.json`` file
```json
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:masters"
    }
  ]
}
```
The group ``system:masters`` membership will give the admin user the power to act as cluster admin when the authorization is enabled.

Create the certificates

    cfssl gencert \
       -ca=ca.pem \
       -ca-key=ca-key.pem \
       -config=cert-config.json \
       -profile=client-authentication \
       admin-csr.json | cfssljson -bare admin

This will produce the ``admin.pem`` certificate file containing the public key and the ``admin-key.pem`` file, containing the private key. Move the key and certificate, along with the Certificate Authority certificate ``ca.pem`` to the client proper location on the client machine and create the ``kubeconfig`` file to access via the ``kubectl`` client
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ca.pem
    server: https://kubernetes:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: default
    user: admin
  name: default/kubernetes/admin
current-context: default/kubernetes/admin
kind: Config
preferences: {}
users:
- name: admin
  user:
    client-certificate: admin.pem
    client-key: admin-key.pem
    username: admin
```

### Tokens
Bearer token authentication is enabled by passing the ``--token-auth-file`` option to API server. The file contains one or more tokens to authenticate user requests presented to the API server. The token file is a csv file with a minimum of 3 columns: token, user name, user uid, followed by optional group names.

For example
```csv
aaaabbbbccccdddd000000000000000a,admin,10000,system:masters
aaabbbbccccdddd0000000000000000b,alice,10001
aaabbbbccccdddd0000000000000000c,joe,10002
```

Move on the client machine and create the ``~/.kube/conf`` file to access via the ``kubectl`` client

    kubectl config set-credentials alice \
      --token=aaaabbbbccccdddd000000000000000b

    kubectl config set-cluster kubernetes \
      --server=https://kubernetes:6443 \
      --certificate-authority=/etc/kubernetes/pki/ca.pem \
      --embed-certs=true

    kubectl config set-context default/kubernetes/alice \
      --cluster=kubernetes \
      --namespace=default \
      --user=alice
      
    kubectl config use-context default/kubernetes/alice

The context file will look like this

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS ...
    server: https://kubernetes:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: default
    user: alice
  name: default/kubernetes/alice
current-context: default/kubernetes/alice
kind: Config
preferences: {}
users:
- name: alice
  user:
    token: aaaabbbbccccdddd000000000000000a
    username: alice
```

When using a token authentication fron an HTTP(S) client, put the token in the request header

    curl -k -H "Authorization:Bearer aaaabbbbccccdddd000000000000000a" https://kubernetes:6443

*Note: the token file cannot be changed without restarting API server.*

### Password
Basic password authentication is enabled by passing the ``--basic-auth-file`` option to API server. The password file is a csv file with a minimum of 3 columns: password, user name, user id and an optional fourth column containing group names. 

For example
```csv
Str0ngPa55word123!,admin,10000,system:masters
Str0ngPa55word456!,alice,10001
Str0ngPa55word789!,joe,10002
```

When using basic authentication from an http client, the API server expects an authorization header with a value of encoded user and password

    BASIC=$(echo -n 'admin:Str0ngPa55word123!' | base64)
    curl -k -H "Authorization:Basic "$BASIC https://kubernetes:6443

*Note: the passwords file cannot be changed without restarting API server.*

### Other methods
Please, see the kubernetes documentation for other authentication methods. 

## Authorization
In kubernetes, all requests to the API server are evaluated against authorization policies and then allowed or denied. All parts of an API request must be allowed by some policy in order to proceed. This means that permissions are denied by default with HTTP status code 403.

Multiple methods can be used at same time, depending on the use case. Most used authorization modes are

  * **Node Authorization**: special-purpose authorizer that grants permissions to kubelet instances running on the worker nodes
  * **Role Based Access Control**: a general-purpose authorizer that regulates access to cluster resources based on the roles of individual users
  
## Node Authorization
The Node Authorization is enabled by passing the ``--authorization-mode=Node`` option to API server. In order to be authorized by the Node authorizer, kubelets must use an authentication credential that identifies them as being in the ``system:nodes`` group, with a username of ``system:node:<nodename>``.

Once a kubelet is authorized by the Node authorizer, it can join the cluster. For kubelets outside the ``system:nodes`` group or without the ``system:node:<nodename>`` name format, the authorization is not performed by the Node authorizer and it will be authorized via other authorize methods, if any. In case the kubelet is not authorized, it cannot join the cluster.

For example, for each worker node identified by ``${nodename}``, create an authentication signing request ``${nodename}-csr.json`` file
```json
{
  "CN": "system:node:${nodename}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "system:nodes"
    }
  ]
}
```
The group ``system:nodes`` membership will give the outhorization by the Node authorizer when the authorization is enabled.

Create the certificates

    cfssl gencert \
      -ca=ca.pem \
      -ca-key=ca-key.pem \
      -config=cert-config.json \
      -profile=client-authentication \
      ${nodename}-csr.json | cfssljson -bare ${nodename}

This will produce the ``${nodename}.pem`` certificate file containing the public key and the ``${nodename}-key.pem`` file, containing the private key. Move the key and certificate, along with the Certificate Authority certificate ``ca.pem`` to the kubelet proper location on the worker node and create the ``kubeconfig`` file as reported [here](./secure.md#securing-the-worker).

The Node authorizer allows a kubelet to perform API operations, including read and write on the kubernetes objects. To limit the API objects kubelet are able to write, enable the Node Restriction admission control plugin by starting the apiserver with the ``--admission-control=NodeRestriction`` option.

### Role Based Admission Control
The Role Based Access Control Authorization is enabled by passing the ``--authorization-mode=RBAC`` option to API server. It is general-purpose authorizer that regulates a given type of access, i.e. get, update, list, delete, to a given resource based on the roles of the individual user or the group the user is belonging to.

#### Roles
In RBAC, a role contains rules that represent a set of permissions. A role can be defined within a namespace with a **Role** object, granting access to resources within that single namespace. For example, the ``kube-system-viewer-role.yaml`` configuration file defines a role for pods viewer in the ``kube-system`` namespace
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: kube-system
  name: kube-system-pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

A role is bound to a particular user or group of users by creating a **RoleBinding** object. For example, the ``kube-system-viewer-role-binding.yaml`` configuration file gives the user ``adriano`` the role of pod viewer in the ``kube-system`` namespace
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-system-pod-viewer
  namespace: kube-system
subjects:
- kind: User
  name: adriano
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: kube-system-pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Create the role and role binding logged as admin user with cluster admin powers

    kubectl create -f kube-system-viewer-role.yaml
    kubectl create -f kube-system-viewer-role-binding.yaml

Login as ``adriano`` user and list the pods in the ``kube-system`` namespace

    kubectl get pods -n kube-system
    NAME                                        READY     STATUS    RESTARTS   AGE
    kube-dns-7797cb8758-5nxm2                   3/3       Running   0          3d
    kube-dns-7797cb8758-7w8tr                   3/3       Running   0          3d
    kube-dns-7797cb8758-nxmzr                   3/3       Running   0          3d
 
Try to delete one of these pods

      kubectl delete pod kube-dns-7797cb8758-5nxm2 -n kube-system
      
      Error from server (Forbidden): pods "kube-dns-7797cb8758-5nxm2" is forbidden:
      User "adriano" cannot delete pods in the namespace "kube-system"

Deleting a pod is not allowed.

A role can be bound to all users belonging to a given group. For example, the following ``kube-system-viewer-role-binding-all-authenticated.yaml`` configuration file gives all authenticad users to be pod viewer in the ``kube-system`` namespace
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kube-system-pod-viewer-all
  namespace: kube-system
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: kube-system-pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

#### Cluster Roles
A **ClusterRole** object can be used to grant the permissions at cluster level, instead of single namespace. They can also be used to grant access to cluster-scoped resources, like nodes, or namespaced resources, like pods, across all namespaces as in ``kubectl get pods --all-namespaces`` command.

For example, the following ``nodes-viewer-cluster-role.yaml`` configuration file create a cluster-scoped role for getting worker nodes in the cluster
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: cluster:nodes-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```

The role above can be assigned, for example, to all authenticated users as defined in the ``nodes-viewer-cluster-role-binding.yaml`` configuration file
```yaml
kind: ClusterRoleBinding
metadata:
  name: cluster:nodes-viewer
subjects:
- kind: Group
  name: system:authenticated
roleRef:
  kind: ClusterRole
  name: cluster:nodes-viewer
  apiGroup: rbac.authorization.k8s.io
```

Create the cluster role and the binding logged as admin user with cluster admin powers

    kubectl create -f nodes-viewer-cluster-role.yaml
    kubectl create -f nodes-viewer-cluster-role-binding.yaml

Login as ``adriano`` user and list the worker nodes

    kubectl get nodes
    NAME      STATUS    ROLES     AGE       VERSION
    kubew03   Ready     <none>    3d        v1.9.2
    kubew04   Ready     <none>    22h       v1.9.2
    kubew05   Ready     <none>    22h       v1.9.2
 
Try to delete one of these nodes

      kubectl delete node kubew03
      
      Error from server (Forbidden): nodes "kubew03" is forbidden:
      User "adriano" cannot delete nodes at the cluster scope

Deleting a node is not allowed.

In any kubernetes cluster, there are some roles and bindings already set

    kubectl get clusterroles
    NAME                                                                   AGE
    admin                                                                  3d
    cluster-admin                                                          3d
    edit                                                                   3d
    viewer                                                                 3d
    system:kube-controller-manager                                         3d
    system:kube-dns                                                        3d
    system:kube-scheduler                                                  3d
    system:node                                                            3d
    ...

    kubectl get clusterrolebindings
    NAME                                           AGE
    cluster-admin                                  3d
    system:kube-controller-manager                 3d
    system:kube-dns                                3d
    system:kube-scheduler                          3d
    system:node                                    3d
    ...

The ``cluster-admin`` cluster scoped role above defines the cluster admin powers. It is bound by default to all users belonging to the ``system:masters`` group. This means that any authenticated user belonging to that group will have cluster admin powers.

The ``cluster-admin`` role above can be used also to define a namespace admin user by bounding it to a given namespace and a given user, as in the following ``tenant-admin-role-binding.yaml`` configuration file
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: tenant:admin
  namespace: project
subjects:
- kind: User
  name: adriano
  namespace: project
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

In the above, the user ``adriano`` will have admin powers only for the ``project`` namespace. It gives full control over every resource in that namespace, including the namespace itself.

To check user's permissions, we'll use the ``kubectl auth can-i`` command test user accounts against the RBAC policies in place. For example, to verify access to pods in a given namespace:

    kubectl auth can-i get pods --namespace project --as adriano
    yes

Do we have access to persistent volumes at cluster level?

    kubectl auth can-i create pv
    no - Required "container.persistentVolumes.create" permission

## Admission Control
Many advanced features in Kubernetes require an **Admission Control** plugin to be enabled in order to properly support the feature. An admission control plugin intercepts requests to the API server after the request is authenticated and authorized. Each admission control plugin is run in sequence before a request is accepted by the API server. If any of the plugins in the sequence reject the request, the entire request is rejected.

The admission control plugins are enabled by starting the apiserver with the ``--admission-control`` option. A complete list of admission control plugins are available on the product documentation.


## Accessing the APIs server
Service accounts are useful when the pod needs to access the pod metadata stored in the API server. For example, the following ``nodejs-pod-namespace.yaml`` definition file implements an API call to read the pod namespace and put it into a pod env variable
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-web-app
  namespace:
  labels:
    app:nodejs
spec:
  containers:
  - name: nodejs
    image: kalise/nodejs-web-app:latest
    ports:
    - containerPort: 8080
    env:
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MESSAGE
      value: "Hello $(POD_NAMESPACE)"
  serviceAccount: default
```

Create the pod above and access the pod

    kubectl create -f nodejs-pod-namespace.yaml

    kubectl get pod -l app=nodejs -o wide
    NAME             READY     STATUS    RESTARTS   AGE       IP          NODE
    nodejs-web-app   1/1       Running   0          13m       10.38.4.9   kubew04

    curl 10.38.4.9:8080
    <html><head></head><body>Hello project from 10.38.4.9</body></html>

we get an answer from the pod being in the namespace name ``project``.

Pods use service accounts to access the following information:

  * name of the pod
  * IP address of the pod
  * labels of the pod
  * annotations of the pod
  * namespace the pod belongs to
  * name of the node the pod is running on
  * name of the service account used by the pod 
  * CPU and memory requests for each container
  * CPU and memory limits for each container

A pod uses exactly one Service Account belonging to the same namespace, but multiple pods inside the same namespace can use the same Service Account.

Service account can also be used to access the APIs server content when working inside a pod. Create a pod from the ``curl.yaml`` file and login into it

    kubectl create -f curl.yaml
    kubectl exec -it curl sh
    / # 

Find the service address of the APIs server by checking the env variables

    / # env | grep KUBERNETES_SERVICE
    KUBERNETES_SERVICE_PORT=443
    KUBERNETES_SERVICE_PORT_HTTPS=443
    KUBERNETES_SERVICE_HOST=10.32.0.1

Trying to access the APIs server from the pod

    / # curl https://$KUBERNETES_SERVICE_HOST:443/version
    curl: (60) SSL certificate problem: unable to get local issuer certificate

we get an error because we're not able to verify the identity of the APIs server. To verify we are talking to the right APIs server and not to a faker, we need to check if the server’s certificate is signed by the known Certification Authority (CA).

    / # ls -l /var/run/secrets/kubernetes.io/serviceaccount/
    total 0
    lrwxrwxrwx    1 root     root            13 Mar 19 15:39 ca.crt -> ..data/ca.crt
    lrwxrwxrwx    1 root     root            16 Mar 19 15:39 namespace -> ..data/namespace
    lrwxrwxrwx    1 root     root            12 Mar 19 15:39 token -> ..data/token

The ``ca.crt`` file above holds the certificate of the Certificate Authority (CA) used to sign the kubernetes API server’s certificate

    / # export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    / # curl https://$KUBERNETES_SERVICE_HOST:443/version
    {
      "kind": "Status",
      "apiVersion": "v1",
      "metadata": {

      },
      "status": "Failure",
      "message": "forbidden: User \"system:anonymous\" cannot get path \"/version\"",
      "reason": "Forbidden",
      "details": {

      },
      "code": 403
    }

We made some progress but still not authenticated by the APIs server. To be authenticate, we'll use the service account token signed by the controller manager via the key specified ``--service-account-private-key-file`` option. That token is mounted as secret into each container

    / # TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
    / # curl -H "Authorization: Bearer $TOKEN" https://$KUBERNETES_SERVICE_HOST:443/version
    {
      "major": "1",
      "minor": "9",
      "gitVersion": "v1.9.2",
      "gitCommit": "5fa2db2bd46ac79e5e00a4e6ed24191080aa463b",
      "gitTreeState": "clean",
      "buildDate": "2018-01-18T09:42:01Z",
      "goVersion": "go1.9.2",
      "compiler": "gc",
      "platform": "linux/amd64"
    }

Now we're authenticated by the APIs server.
