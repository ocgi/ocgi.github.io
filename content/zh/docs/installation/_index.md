---
weight: 2
bookFlatSection: true
title: "Installation"
---

OCGI的组件，主要包括`Carrier` controller，`GPA` controller，`cost-server`等。

执行 `kubectl apply -f https://raw.githubusercontent.com/ocgi/install/main/yaml/ocgi.yaml`
可以快速部署，单独部署组件可以参考本文中其余内容。

> 文档中的所有示例都在[TKE](https://cloud.tencent.com/product/tke)中运行过。

## 部署Carrier controller

* 创建CRD

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/carrier/master/manifeasts/crd.yaml
customresourcedefinition.apiextensions.k8s.io/gameservers.carrier.ocgi.dev created
customresourcedefinition.apiextensions.k8s.io/gameserversets.carrier.ocgi.dev created
customresourcedefinition.apiextensions.k8s.io/squads.carrier.ocgi.dev created
customresourcedefinition.apiextensions.k8s.io/webhookconfigurations.carrier.ocgi.dev created

# kubectl get crd
NAME                                    CREATED AT
gameservers.agones.dev                  2020-11-08T04:07:32Z
gameservers.carrier.ocgi.dev            2020-11-23T06:48:59Z
gameserversets.carrier.ocgi.dev         2020-11-23T06:48:59Z
webhookconfigurations.carrier.ocgi.dev  2020-11-23T06:48:59Z
...
```

* 部署Carrier

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/carrier/master/manifeasts/deploy.yaml 
serviceaccount/carrier created
clusterrolebinding.rbac.authorization.k8s.io/carrier created
deployment.apps/carrier created
```

* 部署carrier-webhook

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/carrier-webhook/master/manifeasts/webhook.yaml
serviceaccount/carrier-webhook created
clusterrole.rbac.authorization.k8s.io/carrier-webhook created
clusterrolebinding.rbac.authorization.k8s.io/carrier-webhook created
service/carrier-webhook-service created
secret/carrier-wbssecret created
deployment.apps/carrier-webhook created
#
# kubectl apply -f https://raw.githubusercontent.com/ocgi/carrier-webhook/master/manifeasts/webhookconfig.yaml 
mutatingwebhookconfiguration.admissionregistration.k8s.io/carrier-mutator created
```

## 部署GPA(General Pod Autoscaler)

* 创建CRD

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/general-pod-autoscaler/master/manifeasts/crd.yaml
customresourcedefinition "generalpodautoscalers.autoscaling.ocgi.dev" created
```

* 部署GPA

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/general-pod-autoscaler/master/manifeasts/gpa.yaml
serviceaccount "gpa" created
clusterrole "gpa" created
clusterrolebinding "gpa" created
deployment "gpa" created
secret "gpa-secret" created
#
# kubectl apply -f https://raw.githubusercontent.com/ocgi/general-pod-autoscaler/master/manifeasts/validatorconfig.yaml
mutatingwebhookconfiguration.admissionregistration.k8s.io/gpa-validator created
```

## 部署cost-server(可选)

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/cost-server/master/manifeasts/cost-server.yaml
```

关于`cost-server`更多信息请参考[设置GamerServer缩容的优先级](/zh/docs/guides/squad-scaledown-priority).
