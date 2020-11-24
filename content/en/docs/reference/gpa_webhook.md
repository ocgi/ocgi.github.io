---
title: "GPA Webhook example"
weight: 2
---

## Introduction

`GPA(GeneralPodAutoscaler)` provides a mechanism based on Webhook to auto-scaling workload. E.g:

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

Webhook Server is implemented by application, so that application can control the number of replicas of workload.

## Implement Webhook Server

[This](https://github.com/ocgi/demowebhook) is a Webhook Server example for Squad.

### Webhook Request and Response

Webhook [API](https://github.com/ocgi/general-pod-autoscaler/tree/master/pkg/requests/api.go) defined as follows:

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

1. The fields received by the Webhook server includes `workload name`, `namespace`, `parameters`, `currentReplicas`
.
2. The webhook should return the `AutoscaleResponse` structure based on the actual situation of the auto-scaling, including `scale` and `replicas`. If the `scale` is set to `false`, it means that the current does not need to be scaled.

### Deploy the Webhook Server

We can deploy the Webhook server [in K8s cluster](https://github.com/ocgi/general-pod-autoscaler/blob/master/manifests/kubernetes/demo-webhook.yaml)), or out of the K8s cluster.

### Auto-scaling based on Webhook

We shoud set the [`webhook`](https://github.com/ocgi/generalpodautoscaler/blob/master/examples/webhook.yaml) field of GeneralPodAutoscaler when auto-scaling based on the Webhook mode.
   
1. If webhook server deployed in K8s cluster, we set the `service` field.

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

2. If webhook server deployed out of K8s cluster, we set the `url` field.

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
