# Introduction to cert-manager for Kubernetes
How to create and test an application using Certmanager and Let's Encrypt in a Kubernetes cluster.

## Concepts 

It's important to understand the various concepts and new Kubernetes resources that <br/>
`cert-manager` introduces.

* Issuers [docs](https://cert-manager.io/docs/concepts/issuer/)
* Certificate [docs](https://cert-manager.io/docs/concepts/certificate/)
* CertificateRequests [docs](https://cert-manager.io/docs/concepts/certificaterequest/)
* Orders and Challenges [docs](https://cert-manager.io/docs/concepts/acme-orders-challenges/)

## Installation 

You can find the latest release for `cert-manager` on their [GitHub Releases page](https://github.com/jetstack/cert-manager/) <br/>

For this demo, I will use K8s 1.30 and `cert-manager` [v1.16.2](https://github.com/cert-manager/cert-manager/releases/download/v1.16.2))

```
# get cert-manager 
cd certificate/cert-manager/
curl -LO https://github.com/jetstack/cert-manager/releases/download/v1.16.2/cert-manager.yaml

# rename file
mv cert-manager.yaml cert-manager-1.16.2.yaml

# install cert-manager 
kubectl apply --validate=false -f cert-manager-1.16.2.yaml
```

## Checking Cert Manager Resources

We can see our components deployed and verify if they are running and not crashing.

```
kubectl -n cert-manager get all

NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-86548b886-2b8x7               1/1     Running   0          77s
pod/cert-manager-cainjector-6d59c8d4f7-hrs2v   1/1     Running   0          77s
pod/cert-manager-webhook-578954cdd-tphpj       1/1     Running   0          77s

NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.96.87.136   <none>        9402/TCP   77s
service/cert-manager-webhook   ClusterIP   10.104.59.25   <none>        443/TCP    77s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE        
deployment.apps/cert-manager              1/1     1            1           77s
deployment.apps/cert-manager-cainjector   1/1     1            1           77s
deployment.apps/cert-manager-webhook      1/1     1            1           77s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-86548b886               1         1         1       77s
replicaset.apps/cert-manager-cainjector-6d59c8d4f7   1         1         1       77s
replicaset.apps/cert-manager-webhook-578954cdd       1         1         1       77

```

## Test Certificate Issuing 

Let's create some test certificates, but in this step I am just testing certmanager and not Let's encrypt.

```
kubectl create ns cert-manager-test

kubectl apply -f ./selfsigned/issuer.yaml

kubectl apply -f ./selfsigned/certificate.yaml

kubectl describe certificate -n cert-manager-test
kubectl get secrets -n cert-manager-test

kubectl delete ns cert-manager-test
```

## Configuration 

https://cert-manager.io/docs/configuration/

## Ingress Controller

Let's deploy an Ingress controller:
> NOTE: This step is not necessary when creating an RKE2 cluster using Rancher Prime, because it will install NGINX by default.

```
kubectl create ns ingress-nginx

kubectl -n ingress-nginx apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/cloud/deploy.yaml

kubectl -n ingress-nginx get pods

kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 80
kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 443
```

We should be able to access NGINX in the browser and see a `404 Not Found` page: http://localhost/
This indicates there are no routes to `/` and the ingress controller is running

## Setup my DNS
I configured DNS using Digital Ocean name and pointed to one of my kubernetes nodes.

I can log into my DNS provider and point my DNS A record to my IP.

If you are running in the cloud, your Ingress controller and Cloud provider will give you a
public IP and you can point your DNS to that accordingly.

## Create Let's Encrypt Issuer for our cluster

We create a `ClusterIssuer` that allows us to issue certs in any namespace. The `ClusterIssuer` is a global setting and not required to specify a Namespace.

```
kubectl apply -f cert-issuer-nginx-ingress.yaml

# check the issuer
kubectl describe clusterissuer letsencrypt-cluster-issuer

```

## Deploy a pod that uses SSL

```
kubectl apply -f .\kubernetes\deployments\
kubectl apply -f .\kubernetes\services\
kubectl get pods
# deploy an ingress route
kubectl apply -f .\kubernetes\cert-manager\ingress.yaml

```

## Issue Certificate 

```
kubectl apply -f certificate.yaml

# check the cert has been issued 

kubectl describe certificate example-app

# TLS created as a secret
kubectl get secrets
NAME                  TYPE                                  DATA   AGE
example-app-tls       kubernetes.io/tls                     2      84m
```
