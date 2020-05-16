## Introduction

This page is intended to provide steps to run PostgreSQL database on Kubernetes cluster.
We are going to use this PostgreSQL server to store Konga configuration.

Note: The below configuration is used to setup single node of PostgreSQL server which is not recommended for production.


## Prerequisites

* Kubernetes Cluster / Minikube 

You can provisioned the kubernetes cluster on any public cloud provider like [AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html), [Google](https://cloud.google.com/kubernetes-engine/docs/how-to), [Azure](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal), etc. Or you can provisioned self managed Kubernetes cluster/minikube.

* Basic understanding of Docker and Kubernetes

You will find many documentations/videos available online to grab the knowledge of Docker and Kubernetes.

#### To Deploy PostgreSQL on Kubernetes cluster, complete the following steps :

* Create Konga Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: konga
```  

* Create Secret to store PostgreSQL configuration. 

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pg-creds
  namespace: konga
type: Opaque
data:
  POSTGRES_DB: konga
  POSTGRES_USER: admin
  POSTGRES_PASSWORD: admin123
```

* Create Persistent Volume Storage for storing DB data.
```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: postgres-pv
  namespace: konga
  labels:
    type: local
    app: postgres
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/datastore/db-data"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pv-claim
  namespace: konga
  labels:
    app: postgres
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
``` 

* Deploy PostgreSQL Server.
 
 I am using postgreSQL 9.6 version, deploy version of your requirement.  
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: postgres
  namespace: konga
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:9.6
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: pg-creds
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim
```
* Create Kubernetes Service object to access PostgreSQL server.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: konga
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
      targetPort: 5432
      protocol: TCP
  selector:
    app: postgres
```

* You can confirm the PostgreSQL server configuration using below commands

```bash
# pod state is running or not
$ kubectl -n konga get pods --no-headers -l app=postgres 
postgres-54858cb448-8xv94   1/1   Running   0     16h

$ kubectl -n konga exec -it  \
$(kubectl -n konga get pods --no-headers -l app=postgres | awk '{print $1}') -- psql --username admin konga
psql (9.6.17)
Type "help" for help.
konga=# \q
```

* Delete the PostgreSQL Deployment
```bash
$ kubectl delete ns kong 
```

#####You can find all the above configuration in a single [postgres.yaml](./postgres.yaml) file.