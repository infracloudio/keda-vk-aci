# keda-vk-aci

Magicscaling event pipelines with Keda and & Virtual Kubelet

Table of Contents
=================

   * [keda-vk-aci](#keda-vk-aci)
   * [Azure Kubernetes Service](#azure-kubernetes-service)
      * [Create a AKS cluster](#create-a-aks-cluster)
   * [Virtual Kubelet](#virtual-kubelet)
      * [Setup Virtual Kubelet](#setup-virtual-kubelet)
   * [Keda](#keda)
      * [Setup Keda](#setup-keda)
   * [Putting it all together](#putting-it-all-together)
      * [Install NATS](#install-nats)
      * [Exposing NATS using Ingress Controller](#exposing-nats-using-ingress-controller)
      * [Create Subscriber deployment](#create-subscriber-deployment)
      * [Create Keda ScaledObject](#create-keda-scaledobject)
      * [Create Publisher deployment](#create-publisher-deployment)
   * [Debugging/Common issues](#debuggingcommon-issues)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

# Azure Kubernetes Service

## Create a AKS cluster

First create a resource group (You can also use an existing one)
```
$ az group create -l southeastasia -n keda-aci-demo
{
  "id": "/subscriptions/904efe62-471e-4637-ac0a-09709c63372d/resourceGroups/keda-aci-demo",
  "location": "southeastasia",
  "managedBy": null,
  "name": "keda-aci-demo",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

Then create a service principal for usage with K8S cluster

```

$ az ad sp create-for-rbac --skip-assignment --name keda-aci-k8s
Changing "keda-aci-k8s" to a valid URI of "http://keda-aci-k8s", which is the required format used for service principal names
{
  "appId": "CLIENT_ID",
  "displayName": "keda-aci-k8s",
  "name": "http://keda-aci-k8s",
  "password": "YOUR_TOP_SECRET_PASSWORD",
  "tenant": "TENANT_ID"
}
```


Finally create a cluster with 1/2 nodes.
```
$ az aks create --resource-group keda-aci-demo -n keda-aci-k8s --ssh-key-value ~/.ssh/id_rsa.pub --node-count 2 --subscription 904efe62-471e-4637-ac0a-09709c63372d --service-principal CLIENT_ID --node-vm-size Standard_D2_v2 --client-secret "YOUR_TOP_SECRET_PASSWORD"

$ az aks get-credentials -g keda-aci-demo -n keda-aci-k8s

$ kubectl cluster-info

```

We willl need master address from the cluster-info results later on.

# Virtual Kubelet

## Setup Virtual Kubelet 

**This will only work in a region where ACI service is offered - please check documentation**

Set all parameters from output of previous commands

```
export ACI_REGION=southeastasia
export AZURE_RG=keda-aci-demo
export AZURE_TENANT_ID=TENANT_ID
export AZURE_CLIENT_ID=CLIENT_ID
export AZURE_CLIENT_SECRET="YOUR_TOP_SECRET_PASSWORD"
export VK_RELEASE=virtual-kubelet-latest
export MASTER_URI=MASTER_URI_FROM_KUBECTL_INFO_COMMAND
export RELEASE_NAME=virtual-kubelet
export VK_RELEASE=virtual-kubelet-latest
export NODE_NAME=virtual-kubelet
export CHART_URL=https://github.com/virtual-kubelet/virtual-kubelet/raw/master/charts/$VK_RELEASE.tgz
export AZURE_SUBSCRIPTION_ID=SUBSCRIPTION_ID
```

And use Helm install:

```
$ helm install "$RELEASE_NAME" \
  --set provider=azure \
  --set providers.azure.targetAKS=true \
  --set providers.azure.masterUri=$MASTER_URI \
  "$CHART_URL"

NAME: virtual-kubelet
LAST DEPLOYED: Sat Jun 13 22:18:48 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
The virtual kubelet is getting deployed on your cluster.

To verify that virtual kubelet has started, run:


  kubectl --namespace=default get pods -l "app=virtual-kubelet"


Note:
TLS key pair not provided for VK HTTP listener. A key pair was generated for you. This generated key pair is not suitable for production use.
```

# Keda

## Setup Keda

```
$ helm repo add kedacore https://kedacore.github.io/charts
$ kubectl create namespace keda
$ helm install keda kedacore/keda --namespace keda
```

# Putting it all together

We will use some of the code and chart from Keda's sample repo which uses NATS with Keda: https://github.com/balchua/gonuts

## Install NATS

```
$ kubectl create namespace stan
$ helm install stan --namespace stan natss-chart
```

Now this NATS can not be accessed by ACI instances due to following open issues, so we will use ingress controller to open NATS on a public Load Balancer and use it for now

- https://github.com/virtual-kubelet/virtual-kubelet/issues/641 
- https://github.com/virtual-kubelet/azure-aci/issues/46

## Exposing NATS using Ingress Controller

```
$ helm install nginx-ingress stable/nginx-ingress --set tcp.4222="stan/stan-nats-ss:4222" --set tcp.8222="stan/stan-nats-ss:8222"

$ kubectl get service nginx-ingress-controller
nginx-ingress-controller        LoadBalancer   10.0.216.254   xx.43.191.xx   80:31403/TCP,443:31456/TCP,4222:32160/TCP,8222:31880/TCP   7h16m
```

Take that public IP of load balancer and replace in following files:

- stan_scaledobject.yaml
- sub-deploy.yaml
- pub-deploy.yaml

## Create Subscriber deployment

```
$ kubectl create -f sub-deploy.yaml 
```

This will create a deployment with one replica

## Create Keda ScaledObject

```
$ kubectl create -f stan_scaledobject.yaml
```

You will notice that Keda creates a HPA and also scales down the deployment to zero so we don't see any pod but deployment exists.

```
$ kubectl get deployment
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
gonuts-sub                        0/0     0            0           7h17m

$ kubectl get hpa
NAME                  REFERENCE               TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-gonuts-sub   Deployment/gonuts-sub   <unknown>/10 (avg)   1         30        0          7h17m
```

## Create Publisher deployment

```
$ kubectl create -f pub-deploy.yaml 
```

Once we create publisher - you will notice that Keda creates many consumer pods.

```
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-5599bb94cf-m99wv          1/1     Running   0          9m6s
nginx-ingress-default-backend-674d599c48-gcntm     1/1     Running   0          9m6s
virtual-kubelet-virtual-kubelet-78d55797cc-5r54m   1/1     Running   0          20h
gonuts-pub-84885779f5-6j6sj                        1/1     Running    0          23m
gonuts-sub-985c75b49-q8tkl                         1/1     Running    0          18s
gonuts-sub-985c75b49-ts4dt                         1/1     Running    0          0s
gonuts-sub-985c75b49-vcr49                         0/1     Pending    0          0s
gonuts-sub-985c75b49-mrx5r                         0/1     Pending    0          0s
gonuts-sub-985c75b49-dqhmc                         0/1     Pending    0          0s
gonuts-sub-985c75b49-wvkq9                         0/1     Pending    0          0s
gonuts-sub-985c75b49-s5ngr                         0/1     Pending    0          0s
```

# Debugging/Common issues

## TBD