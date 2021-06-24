---
weight: 2
bookFlatSection: true
title: "Installation"
---

The components of OCGI mainly include `Carrier` controller, `GPA` controller, and `cost-server`, etc.

Quick install by running `kubectl apply -f https://raw.githubusercontent.com/ocgi/install/main/yaml/ocgi.yaml`.
If you would like install a single component, please refer to the following docs.

> We have ran all examples on [TKE](https://cloud.tencent.com/product/tke).
 
## Install Carrier controller

  * Install CRD

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

  * Install Carrier

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/carrier/master/manifeasts/deploy.yaml 
serviceaccount/carrier created
clusterrolebinding.rbac.authorization.k8s.io/carrier created
deployment.apps/carrier created
```

  * Install Carrier-webhook

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

## Install GPA(General Pod Autoscaler)

  * Install CRD

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/general-pod-autoscaler/master/manifeasts/crd.yaml
customresourcedefinition "generalpodautoscalers.autoscaling.ocgi.dev" created
```

  * Install GPA 

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

## Install cost-server(optional)

```shell script
# kubectl apply -f https://raw.githubusercontent.com/ocgi/cost-server/master/manifeasts/cost-server.yaml
```

For more information abount `cost-server`,  please refer to the [`GamerServer scaling down priority`](/en/docs/guides/squad-scaledown-priority).
