---
weight: 2
bookFlatSection: true
title: "Installation"
---

OCGI的组件，主要包括`Carrier` controller，`GPA` controller，`cost-server`等。

> 文档中的所有示例都在[TKE](https://cloud.tencent.com/product/tke)中运行过。

## 部署Carrier controller

  * 创建CRD

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

  * 部署Carrier

```shell script
# kubectl apply -f https://github.com/ocgi/carrier/blob/master/deployment/deploy.yaml 
serviceaccount/carrier created
clusterrolebinding.rbac.authorization.k8s.io/carrier created
deployment.apps/carrier created
```

  * 创建SDK service account

  SDK-Sever作为Sidecar自动注入到应用Pod，并访问kube-apiserver。

```shell script
# kubectl apply -f https://github.com/ocgi/carrier-sdk/blob/master/manifeasts/serviceaccount.yaml
clusterrole.rbac.authorization.k8s.io/carrier-sdk created
serviceaccount/carrier-sdk created
rolebinding.rbac.authorization.k8s.io/carrier-sdk-access created
```

  * 部署carrier-webhook

  TODO.

## 部署GPA(General Pod Autoscaler)

  * 创建CRD
  
```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/config/crd.yaml
customresourcedefinition "generalpodautoscalers.autoscaling.ocgi.dev" created
```

  * 部署GPA 

```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/deployment/gpa.yaml
serviceaccount "gpa" created
clusterrolebinding "gpa" created
deployment "gpa" created
```

  * 部署gpa-webhook(optional, only for webhook mode, it can keep buffer for gameservers)


```shell script
# kubectl apply -f https://github.com/ocgi/general-pod-autoscaler/blob/master/deployment/gpa-webhook.yaml
deployment "gpa-webhook" created
service "gpa-webhook" created
```

## 部署cost-server(可选)

```shell script
# kubectl apply -f https://github.com/ocgi/cost-server/blob/master/manifeasts/cost-server.yaml
```

关于`cost-server`更多信息请参考[设置GamerServer缩容的优先级](/zh/docs/guides/squad-scaledown-priority).
