 ![alt text](https://blog.ptrk.io/content/images/size/w2000/2018/04/cover-1.png)

 # How to deploy EFK logging in Kubernetes cluster
 
 When running multiple services and applications on a Kubernetes cluster, a centralized, cluster-level logging stack can help you quickly sort through and analyze the heavy volume of log data produced by your Pods. One popular centralized logging solution is the Elasticsearch, Fluentd, and Kibana (EFK) stack.

 ## Prerequisites

Before you begin with this guide, ensure you have the following available to you:

A Kubernetes 1.10+ cluster with role-based access control (RBAC) enabled

Ensure your cluster has enough resources available to roll out the EFK stack, and if not scale your cluster by adding worker nodes. We’ll be deploying a 3-Pod Elasticsearch cluster (you can scale this down to 1 if necessary), as well as a single Kibana Pod. Every worker node will also run a Fluentd Pod. The cluster in this guide consists of 3 worker nodes and a managed control plane.

The kubectl command-line tool installed on your local machine, configured to connect to your cluster. You can read more about installing kubectl in the official documentation.

Once you have these components set up, you’re ready to begin with this guide.

## Creating a Namespace
Before we roll out an Elasticsearch cluster, we’ll first create a Namespace into which we’ll install all of our logging instrumentation.

```
kubectl create namespace kube-logging
```

## Ingress Deployment

The Ingress resource embodies this idea, and an Ingress controller is meant to handle all the quirks associated with a specific "class" of Ingress.

An Ingress Controller is a daemon, deployed as a Kubernetes Pod, that watches the apiserver's /ingresses endpoint for updates to the Ingress resource. Its job is to satisfy requests for Ingresses.

The following Mandatory Command is required for all deployments.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

## minikube

```
minikube addons enable ingress
```

## AWS

In AWS we use an Elastic Load Balancer (ELB) to expose the NGINX Ingress controller behind a Service of Type=LoadBalancer. Since Kubernetes v1.9.0 it is possible to use a classic load balancer (ELB) or network load balancer (NLB).

Elastic Load Balancer - ELB
This setup requires to choose in which layer (L4 or L7) we want to configure the ELB:

Layer 4: use TCP as the listener protocol for ports 80 and 443.
Layer 7: use HTTP as the listener protocol for port 80 and terminate TLS in the ELB
For L4:

Check that no change is necessary with regards to the ELB idle timeout. In some scenarios, users may want to modify the ELB idle timeout, so please check the ELB Idle Timeouts section for additional information. If a change is required, users will need to update the value of service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout in provider/aws/service-l4.yaml

Then execute:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/service-l4.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/patch-configmap-l4.yaml
```
For L7:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/service-l7.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/aws/patch-configmap-l7.yaml
```
## Now Deploy EFK

### Elasticsearch

Create a elasticsearch service

```
kubectl create -f elasticsearch_svc.yaml
```

Creating elastic search StatefulSet

```
kubectl create -f elasticsearch_statefulset.yaml
```
You can monitor the StatefulSet as it is rolled out using kubectl rollout status:

```
kubectl rollout status sts/es-cluster --namespace=kube-logging
```
You should see the following output as the cluster is rolled out:

```
Output
Waiting for 3 pods to be ready...
Waiting for 2 pods to be ready...
Waiting for 1 pods to be ready...
partitioned roll out complete: 3 new pods have been updated...
```
### Creating Kibana Deployment and Service

To launch Kibana on Kubernetes, we’ll create a Service called kibana, and a Deployment consisting of one Pod replica. You can scale the number of replicas depending on your production needs, and optionally specify a LoadBalancer type for the Service to load balance requests across the Deployment pods.

```
kubectl create -f kibana.yaml
```

### Creating the Fluentd DaemonSet

To deploy fludentd run below command

```
kubectl create -f fluentd.yaml
```
Note:- change stroage class and service type as per your requirement, here I am  using storage class gp2 and service as a cluster ip

Now everything is deploy, now we will protect kibana using basic auth

## Ingress Authentication

### Create a User and Password

```
htpasswd -c ./auth kibana
```
It is important to name the file auth. The filename is used as the key in the key-value pair under the data section of secret.

here kibana is user,

After entering a password for the new user twice, you end up with the file auth.

### Create a Secret

```
kubectl create secret generic kibana --from-file auth
```

### Protect Ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: kibana
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required - ok"
spec:
  rules:
  - http:
      paths:
        - path: /
          backend:
            serviceName: kibana
            servicePort: 5601
```
Now apply that

```
kubectl create -f auth-ingress.yaml
```
Now everything is done access your kibana using ingress controller ELB.

[note: create secret and EFK stack in same namespace, here we are using kube-logging ]

## Refernces

https://imti.co/kubernetes-ingress-basic-auth/
https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-elasticsearch-fluentd-and-kibana-efk-logging-stack-on-kubernetes

