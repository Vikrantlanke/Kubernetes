## Introduction

This page is intended to provide steps to run Konga on Kubernetes cluster. Konga is an open source tool that enables you to manage your Kong API Gateway with ease.
It is developed by Panagis Tselentis a software engineer leaves in Amsterdam. I found this tool really handy for managing Kong APIs through UI. To know more about konga visit [github](https://github.com/pantsel/konga).

## Prerequisites

* Kubernetes Cluster / Minikube 

You can provisioned the kubernetes cluster on any public cloud provider like [AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html), [Google](https://cloud.google.com/kubernetes-engine/docs/how-to), [Azure](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal), etc. Or you can provisioned self managed Kubernetes cluster/minikube.

* Basic understanding of Docker and Kubernetes

You will find many documentations/videos available online to grab the knowledge of Docker and Kubernetes.

* Database Servers to store its configuration (MySQL, PostgreSQL, MongoDB). I have used PostgreSQL as a backend you can find its documentation [here](./Postgres-Server.md).

#### To Deploy Konga on Kubernetes cluster, complete the following steps :

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

Note: The above to steps are already covered in PostgreSQL documentation, skip it if you already did above steps. 

* Deploy Konga Server

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: konga
  namespace: konga
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: konga
        app: konga
    spec:
      containers:
      - name: konga
        image: pantsel/konga
        env:
        - name: DB_ADAPTER
          value: postgres
        - name: DB_HOST
          value: postgres
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: pg-creds
              key: POSTGRES_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: pg-creds
              key: POSTGRES_PASSWORD
        - name: DB_DATABASE
          valueFrom:
            secretKeyRef:
              name: pg-creds
              key: POSTGRES_DB
        ports:
        - containerPort: 1337
```
* Create Kubernetes Service object for Konga.
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: konga
  name: konga-svc
  namespace: konga
spec:
  ports:
    - protocol: TCP
      port: 443
      targetPort: 1337
  selector:
    app: konga
``` 
* Create Ingress rule to access Konga outside the Kubernetes Cluster. I did this deployment on Azure AKS with Azure Load Balancer, Azure Private DNS and Nginx Ingress Controller.
You can deploy it any public cloud just make sure you have prerequisites configured on your cluster. 

```yaml
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  name: konga
  namespace: konga
spec:
  rules:
  - host: konga.example.com
    http:
      paths:
      - backend:
          serviceName: konga-svc
          servicePort: 443
        path: /
  tls:
    - hosts:
      - konga.example.com
      secretName: certificate-secret
``` 
Note: You need to change your DNS record name with konga.example.com and also the valid certificate secret object for your domain.

#####You can find all the above configuration in a single [konga.yaml](./konga.yaml) file.