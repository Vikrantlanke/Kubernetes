## Introduction

This page is intended to provide steps to run Kong on Kubernetes cluster. 
Kong is most popular open source tool that enables you to manage the traffic flowing from external world to your application usinng multiple plugins like rate limiting, consumer authentication, request/response size limiting, etc.
It is build on light weight proxy Nginx and the newer version of Kong application gateway comes along with kong ingress controller which gives more power to control the traffic.
Newer version of kong supports imperative configuration as well as declarative configuration. 
Imperative Configuration means Kong configuration stored in Database like PostgreSQL and declarative configuration means Kong configuration stored in main memory of the cluster.  


In this page, I’ll show some of the basic commands and declarative configurations as well as example of how to use Kong with applications

## Prerequisites

* Kubernetes Cluster / Minikube 

You can provisioned the kubernetes cluster on any public cloud provider like [AWS](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html), [Google](https://cloud.google.com/kubernetes-engine/docs/how-to), [Azure](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal), etc. Or you can provisioned self managed Kubernetes cluster/minikube.

* Basic understanding of Docker and Kubernetes

You will find many documentations/videos available online to grab the knowledge of Docker and Kubernetes.

##### Note: I followed the official kong [documentation](https://docs.konghq.com/2.0.x/kong-for-kubernetes/install/) to setup kong but done some twicks which may help you to setup kong smoothly. I did the installation on Azure AKS cluster, you can go for any other cloud provider.

#### To Deploy Kong on Kubernetes cluster, complete the following steps :

* Install kong configuration 
```bash
$ kubectl apply -f ./kong-ingress-dbless.yaml
```

* Kong Ingress rule to expose kong proxy and kong admin 
```bash
$ kubectl apply -f ./kong-ingress-rule.yaml
```
Note: You need to change your DNS record name with kong.example.com & kongadmin.example.com

* To test the Kong deployment, curl both the endpoints of kong proxy and kong admin.  
```bash
$ curl -i http://kongadmin.example.com/metrics/
# You suppose to see the default metrics expose by kong for prometheus

$ curl -i http://kong.example.com/
HTTP/1.1 404 Not Found
Date: Sat, 16 May 2020 10:58:01 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/2.0.4

{"message":"no Route matched with those values"}
```

* Now to deploy the sample application and test it. 

```bash
$ kubectl apply -f echo-server.yaml
$ kubectl apply -f echo-ingress-rule.yaml

$ curl -i http://echo.example.com
```

* Lets do some magic with kong plugins to limit and authenticate traffic with sample application.


Configure Rate Limiting plugin to allow 5 request per minute. 
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit
config:
    minute: 5
    policy: local
plugin: rate-limiting
```

Configure Auth Key plugin to validate the request for the application
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: key-auth
config:
  key_names:
    - api_key
enabled: true
plugin: key-auth
```

You can attach both the plugins to Route (Ingress rule) or Service (application service) with annotations. 
Below example shows how to attach it with Route. Change the existing [file](echo-ingress-rule.yaml) of application ingress.
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: api
  labels:
    app: api
  annotations:
    plugins.konghq.com: rate-limit, key-auth
spec:
  rules:
    - host: echo.example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 80
```

Lets configure a consumer who will consume sample application with a valid Key. 
```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: test-user
username: test-user

---
apiVersion: configuration.konghq.com/v1
kind: KongCredential
metadata:
  name: user-credential
consumerRef: test-user
type: key-auth
config:
  key: 62eb165c070a41d5c1b58d9d3d725ca1
```
The open source version of kong supports Basic(username & password), API Keys, HMAC, OAUTH2 & JWT authentication type.
To get the advance authentication options you can go for Enterprise version of the Kong. 

Now test the sample application with consumer key
```bash
$ curl -i -H "api_key: 62eb165c070a41d5c1b58d9d3d725ca1" http://echo.example.com
HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Date: Sat, 16 May 2020 11:33:03 GMT
Server: echoserver
X-RateLimit-Remaining-Minute: 4
X-RateLimit-Limit-Minute: 5
RateLimit-Remaining: 4
RateLimit-Limit: 5
RateLimit-Reset: 57
X-Kong-Upstream-Latency: 1
X-Kong-Proxy-Latency: 0
Via: kong/2.0.4
...
```
