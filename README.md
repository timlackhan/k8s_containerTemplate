# Container Deployment on clearlinux

## Basic Concepts

### PV

Persistent Volume, it you want to get some result data in your pod, you can use pv, which will be mounted on your pod and help you get your data. Here we choose NFS to be the mounted medium.

### PVC

Persistent Volume Claim, it's not enough to have pv only, you should use pvc to claim that I will use pv and after that you can combine pv with your pod.

### RBAC

Role Based Access Control. It controls whether your pod can get [or create...] pods [or services...] in your k8s cluster.

### Service Account 

Just like the group of pod belongs to.

### Role

It determines the rules about what action your pod can do (like kubectl get, or kubectl create..) and what kind of resources your pod can visit (like pods or services...)

### Rolebinding

Role determines what pod can do and rolebinding binds a certain group of pods with role via service account, because service account is just like the group of pod belongs to, you can imagine it as a label.

 

## Architecture Graph

![ContainerTemplate](https://github.com/timlackhan/k8s_containerTemplate/blob/master/ContainerTemplate.png)



## 1. Prepare one master node and one worker node

### 1.1 Create a k8s cluster on two nodes

### 1.2 Create  namespace "containers" on your cluster

1. kubectl create ns containers



## 2. Prepare for PVC

### 2.1 Prepare for NFS on master node

1. mkdir /data/{v1,v2,v3,v4,v5} -pv
2. vi /etc/exports

```
/data/v1 10.239.0.0/16(rw,no_root_squash,no_subtree_check)
/data/v2 10.239.0.0/16(rw,no_root_squash,no_subtree_check)
/data/v3 10.239.0.0/16(rw,no_root_squash,no_subtree_check)
/data/v4 10.239.0.0/16(rw,no_root_squash,no_subtree_check)
/data/v5 10.239.0.0/16(rw,no_root_squash,no_subtree_check)
```

3. exportfs -avr

4. swupd bundle-add nfs-utils

5. systemctl start rpcbind

6. systemctl start nfs-mountd

7. systemctl start nfs-utils

8. showmount -e

```
Export list for v-30390-3:
/data/v5 10.239.0.0/16
/data/v4 10.239.0.0/16
/data/v3 10.239.0.0/16
/data/v2 10.239.0.0/16
/data/v1 10.239.0.0/16
```

ps. 10.239.0.0/16 is subnetwork of VMs

### 2.2 Open NFS on worker node

1. swupd bundle-add nfs-utils

2. systemctl start rpcbind

3. systemctl start nfs-mountd

4. systemctl start nfs-utils

### 2.3 Prepare for PVs on master node

1. vi ~/pv/pv.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv01
  labels:
    name: pv01
spec:
  nfs:
    path: /data/v1/
    server: 10.239.85.46
  accessModes: ["ReadWriteOnce","ReadWriteMany"]
  capacity:
    storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv02
  labels:
    name: pv02
spec:
  nfs:
    path: /data/v2/
    server: 10.239.85.46
  accessModes: ["ReadWriteOnce","ReadWriteMany"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv03
  labels:
    name: pv03
spec:
  nfs:
    path: /data/v3/
    server: 10.239.85.46
  accessModes: ["ReadWriteOnce","ReadWriteMany"]
  capacity:
    storage: 3Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv04
  labels:
    name: pv04
spec:
  nfs:
    path: /data/v4/
    server: 10.239.85.46
  accessModes: ["ReadWriteOnce","ReadWriteMany"]
  capacity:
    storage: 4Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv05
  labels:
    name: pv05
spec:
  nfs:
    path: /data/v5/
    server: 10.239.85.46
  accessModes: ["ReadWriteOnce","ReadWriteMany"]
  capacity:
    storage: 5Gi
```

2. kubectl create -f ~/pv/pv.yaml
3. kubectl get pv

```
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON   AGE
pv01   1Gi        RWO,RWX        Retain           Available                                              29h
pv02   2Gi        RWO,RWX        Retain           Available                                              29h
pv03   3Gi        RWO,RWX        Retain           Available                                              29h
pv04   4Gi        RWO,RWX        Retain           Available                                              29h
pv05   5Gi        RWO,RWX        Retain           Available                                              29h
```

### 2.4 Prepare for PVC on master node

1. vi ~/pv/pvc.yaml

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc01
  namespace: containers
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 1Gi
```

2. kubectl create -f ~/pv/pvc.yaml -n containers

2. kubectl get pvc -n containers

```
NAMESPACE    NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
containers   pvc01   Bound    pv01     1Gi        RWO,RWX                       29h
```

4. kubectl get pv

```
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM              STORAGECLASS   REASON   AGE
pv01   1Gi        RWO,RWX        Retain           Bound       containers/pvc01                           29h
pv02   2Gi        RWO,RWX        Retain           Available                                              29h
pv03   3Gi        RWO,RWX        Retain           Available                                              29h
pv04   4Gi        RWO,RWX        Retain           Available                                              29h
pv05   5Gi        RWO,RWX        Retain           Available                                              29h
```



## 3. Prepare for RBAC

### 3.1 Create service account

1. kubectl create sa foo -n containers
2. kubectl get sa -n containers

```
NAME      SECRETS   AGE
default   1         32h
foo       1         32h
```

### 3.2 Create role

1. vi ~/RBAC/role.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: containers
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get","watch","list"]
  resources: ["pods"]
```

2. kubectl create -f ~/RBAC/role.yaml -n containers
3. kubectl get role -n containers

```
NAME             AGE
service-reader   32h
```

### 3.3 Create rolebinding

1. vi ~/RBAC/rolebinding.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test
  namespace: containers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: foo
  namespace: containers
```

2. kubectl create -f  ~/RBAC/rolebinding.yaml -n containers
3. kubectl get rolebinding -n containers

```
NAME   AGE
test   31h
```



## 4. Prepare for pods

### 4.1 Create pods

1. vi ~/deployment/mongodb.yaml

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: mongodb-deployment
spec:
  replicas: 1
  template:
    metadata:
      name: mongodb-deployment
      labels:
        app: mongodb

    spec:
      serviceAccountName: foo

      containers:
      - image: docker.io/mongo
        name: mongodb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
      - name: ambassador
        image: luksa/kubectl-proxy:1.6.2
      - name: main
        image: tutum/curl
        command: ["sleep", "9999999"]

      volumes:
      - name: mongodb-data
        persistentVolumeClaim:
          claimName: pvc01
```

2. kubectl create -f mongodb.yaml -n containers

### 4.2 Check whether Role has effect on your pod

1. kubectl exec -it mongodb-deployment-9f7dccbfc-rbtzs -c main curl localhost:8001/api/v1/namespaces/**containers/pods** -n containers

```
 ...
 "ready": true,
            "restartCount": 0,
            "image": "docker.io/library/mongo:latest",
            "imageID": "docker.io/library/mongo@sha256:968de6d4e8d28de52158d7ff6744422bd94aedd375df4ea93617cd7491329850",
            "containerID": "cri-o://03ba9a9248e7b23d0590a442a0ab44ef29bfac7bdc0b81dbb1185dee8fbdbc6e"
          }
 ...
```

2. kubectl exec -it mongodb-deployment-9f7dccbfc-rbtzs -c main curl localhost:8001/api/v1/namespaces/**containers/services** -n containers

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:containers:foo\" cannot list resource \"services\" in API group \"\" in the namespace \"containers\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
```

3. kubectl exec -it mongodb-deployment-9f7dccbfc-rbtzs -c main curl localhost:8001/api/v1/namespaces/**default/pods** -n containers

```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "services is forbidden: User \"system:serviceaccount:containers:foo\" cannot list resource \"services\" in API group \"\" in the namespace \"containers\"",
  "reason": "Forbidden",
  "details": {
    "kind": "services"
  },
  "code": 403
```

