# Using Linkerd and with Argo Rollouts

[Linkerd](https://linkerd.io/) is a service mesh for Kubernetes. It makes running services easier and safer by giving you runtime debugging, observability, reliability, and security—all without requiring any changes to your code.

## Prerequisites

A Kubernetes cluster. If you do not have one, you can create one using [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/), or any other Kubernetes cluster. This guide will use Kind.

Linkerd installed in your Kubernetes cluster.


## Step 1 - Create a Kind cluster by running the following command:

```shell
kind delete cluster &>/dev/null
kind create cluster --config ./kind-cluster.yaml
```

## Step 2 - Install Linkerd and Linkerd Viz by running the following commands:

I will use the Linkerd CLI to install Linkerd in the cluster. You can also install Linkerd using Helm or kubectl.
I tested this guide with Linkerd version 2.13.0

```shell
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f - && linkerd check
linkerd viz install | kubectl apply -f - && linkerd check
```


## Step 3 - Install Argo Rollouts and Argo Rollouts plugin to allow Linkerd to manage the traffic:

```shell
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl apply -k https://github.com/argoproj/argo-rollouts/manifests/crds\?ref\=stable
kubectl apply -f argo-rollouts-plugin.yaml
```

## Step 4 - Give access to Argo Rollouts for the Gateway/Http Route

```shell
kubectl apply -f cluster-role.yaml
```
__Note:__ These permission are very permissive. You should lock them down according to your needs.

With the following role we allow Argo Rollouts to have write access to HTTPRoutes and Gateways.

```shell
kubectl apply -f cluster-role-binding.yaml
```
## Step 5 - Create HTTPRoute that defines a traffic split between two services

Create HTTPRoute and connect to the created Gateway resource

```shell
kubectl apply -f httproute.yaml
```
## Step 6 - Create the services required for traffic split 

Create three Services required for canary based rollout stratedy

```shell
kubectl apply -f service.yaml
```

## Step 7 - Create the services required for traffic split 

Add Linkerd annotaions to the namespace where the services are deployed

```shell
kubectl apply -f namespace.yaml
```

## Step 8 - Create an example Rollout

Deploy a rollout to get the initial version.
```shell
kubectl apply -f rollout.yaml
```

## Step 9 - Watch the rollout
```shell
watch "kubectl -n default get httproute.gateway.networking.k8s.io/argo-rollouts-http-route -o custom-columns=NAME:.metadata.name,PRIMARY_SERVICE:.spec.rules[0].backendRefs[0].name,PRIMARY_WEIGHT:.spec.rules[0].backendRefs[0].weight,CANARY_SERVICE:.spec.rules[0].backendRefs[1].name,CANARY_WEIGHT:.spec.rules[0].backendRefs[1].weight"
```