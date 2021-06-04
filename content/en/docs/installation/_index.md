---
weight: 2
bookFlatSection: true
title: "Installation"
---

The components of OCGI mainly include `Carrier` controller, `GPA` controller, and `cost-server`, etc.

> We have ran all examples on [TKE](https://cloud.tencent.com/product/tke).
 
## Install Carrier controller

  * Install CRD

```shell script
# kubectl apply -f https://github.com/ocgi/carrier/blob/master/config/crd.yaml 
customresourcedefinition.apiextensions.k8s.io/gameservers.carrier.ocgi.dev created
customresourcedefinition.apiextensions.k8s.io/gameserversets.carrier.ocgi.dev created
customresourcedefinition.apiextensions.k8s.io/squads.carrier.ocgi.dev created

# kubectl get crd
NAME                                    CREATED AT
gameservers.carrier.ocgi.dev            2020-11-23T06:48:59Z
gameserversets.carrier.ocgi.dev         2020-11-23T06:48:59Z
squads.carrier.ocgi.dev                 2020-11-23T06:48:59Z
...
```

  * Install Carrier

```shell script
# kubectl apply -f https://github.com/ocgi/carrier/blob/master/deployment/deploy.yaml 
serviceaccount/carrier created
clusterrolebinding.rbac.authorization.k8s.io/carrier created
deployment.apps/carrier created
```

  * Install SDK service account

  The service account is uesed by SDK-Sever, which is automatically injected into the application Pod as a Sidecar and accesses kube-apiserver.

```shell script
# kubectl apply -f https://github.com/ocgi/carrier-sdk/blob/master/deployment/serviceaccount.yaml
clusterrole.rbac.authorization.k8s.io/carrier-sdk created
serviceaccount/carrier-sdk created
rolebinding.rbac.authorization.k8s.io/carrier-sdk-access created
```

## Install GPA(General Pod Autoscaler)

  * Install CRD
  
```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/config/crd.yaml
customresourcedefinition "generalpodautoscalers.autoscaling.ocgi.dev" created
```

  * Install GPA 

```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/deployment/gpa.yaml
serviceaccount "gpa" created
clusterrolebinding "gpa" created
deployment "gpa" created
```

  * Install gpa-webhook(optional, only for webhook mode, it can keep buffer for gameservers)

```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/deployment/gpa-webhook.yaml
deployment "gpa-webhook" created
service "gpa-webhook" created
```

## Install cost-server(optional)

```shell script
# kubectl apply -f https://github.com/ocgi/cost-server/blob/master/manifeasts/cost-server.yaml
```

For more information abount `cost-server`,  please refer to the [`GamerServer scaling down priority`](/en/docs/guides/squad-scaledown-priority).
