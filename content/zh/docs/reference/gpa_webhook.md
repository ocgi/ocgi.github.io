---
title: "GPA Webhook示例"
weight: 2
---

## 简介

`GPA(GeneralPodAutoscaler)`提供了基于Webhook的机制来实现Workload的自动伸缩。例如：

```yaml
apiVersion: autoscaling.ocgi.dev/v1alpha1
kind: GeneralPodAutoscaler
metadata:
  name: pa-squad
  namespace: default
spec:
  maxReplicas: 8
  minReplicas: 1
  scaleTargetRef:
    apiVersion: carrier.ocgi.dev/v1alpha1
    kind: Squad
    name: squad-example
  webhook:
    parameters:
      buffer: "2"
    service:
      name: gpa-webhook
      namespace: kube-system
      path: scale
      port: 8000
```

`Webhook Server`由用户自己实现，从而实现由用户来控制Workload的副本数量。

## 实现Webhook Server

[这里](https://github.com/ocgi/demowebhook) 是一个针对Squad开发的Webhook Server示例。

### Webhook Request and Response

Webhook [api](https://github.com/ocgi/general-pod-autoscaler/tree/master/pkg/requests/api.go) 定义如下

```go

// AutoscaleRequest defines the request to webhook autoscaler endpoint
type AutoscaleRequest struct {
	// UID is used for tracing the request and response.
	UID types.UID `json:"uid"`
	// Name is the name of the workload(Squad, Statefulset...) being scaled
	Name string `json:"name"`
	// Namespace is the workload namespace
	Namespace string `json:"namespace"`
	// Parameters are the parameter that required by webhook
	Parameters map[string]string `json:"parameters"`
	// CurrentReplicas is the current replicas
	CurrentReplicas int32 `json:"currentReplicas"`
}

// AutoscaleResponse defines the response of webhook server
type AutoscaleResponse struct {
	// UID is used for tracing the request and response.
	// It should be same as it in the request.
	UID types.UID `json:"uid"`
	// Set to false if should not do scaling
	Scale bool `json:"scale"`
	// Replicas is targeted replica count from the webhookServer
	Replicas int32 `json:"replicas"`
}

// AutoscaleReview is passed to the webhook with a populated Request value,
// and then returned with a populated Response.
type AutoscaleReview struct {
	Request  *AutoscaleRequest  `json:"request"`
	Response *AutoscaleResponse `json:"response"`
}

```

1. Webhook server接收到的信息包括`workload name`, `namespace`, `parameters`，`currentReplicas`

2. Webhook 应该基于需要的扩缩容实际情况应该返回`AutoscaleResponse`结构，包含`scale`和 `replicas`, 如果`scale`设置为false， 代表当前不需要伸缩。

### 部署Webhook Server


[在K8s中部署](https://github.com/ocgi/general-pod-autoscaler/blob/master/manifests/kubernetes/demo-webhook.yaml), 同时也支持不在K8s中部署


### 使用Webhook Mode是扩缩容

使用[webhook 模式进行扩缩容](https://github.com/ocgi/general-pod-autoscaler/blob/master/examples/webhook.yaml), 需要在GPA中填写Webhook Server相关的信息， 如下所示： 
   
如果webhook部署在K8s中, 我们可以添加Service相关的信息到`service`字段

```yaml
    apiVersion: autoscaling.ocgi.dev/v1alpha1
    kind: GeneralPodAutoscaler
    metadata:
      name: pa-test1
    spec:
      maxReplicas: 8
      minReplicas: 2
      scaleTargetRef:
        apiVersion: carrier.ocgi.dev/v1alpha1
        kind: GameServerSet
        name: example
      webhook:
        service:
          namespace: kube-system
          name: demowebhook
          port: 8000
          path: scale
        parameters:
          buffer: "3"   
```

如果webhook不是部署在K8s中, 我们需要填写`service`字段中的`url`

```yaml
    apiVersion: autoscaling.ocgi.dev/v1alpha1
    kind: GeneralPodAutoscaler
    metadata:
      name: pa-test1
    spec:
      maxReplicas: 8
      minReplicas: 2
      scaleTargetRef:
        apiVersion: carrier.ocgi.dev/v1alpha1
        kind: GameServerSet
        name: example
      webhook:
        url: http://123.test.com:8080/scale
        parameters:
          buffer: "3"   
```