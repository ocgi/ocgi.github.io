---
title: "Carrier SDK"
weight: 1
---

## 背景

一般来说，游戏后端server会缓存一些玩家状态数据。在自动弹性伸缩的时候，`Carrier controller`不能随意创建/删除`GameServer`。K8s提供的`Statefulset`，`Deployment`等workload在缩容、变更删除Pod时缺少与应用程序的确认。

而`GameServer workload`提供了一个简单的`SDK`，游戏后端server可以把当前的一些服务状态信息，通知到`Carrier controller`，用于`Carrier controller`在弹性伸缩、或者发布变更时，选择合适的副本进行删除。

## Game server与SDK的关系

![SDK archtecture](/images/sdk_archtecture.jpg)

* `SDK-Server`作为`sidecar`容器(由`Carrier controller`自动注入)，与`GameServer`容器，运行在同一个K8s pod。
* `GameServer`可以通过`SDK API`访问`SDK-Server`。如果应用程序不想调用`SDK API`，可以配置相应的Webhook，由`SDK-Server`来调用Webhook。
* `SDK-Server`连接`K8s API`，并更新`GameServer CRD`。

## 用户自定义Condition

`GameServer`可以通过`SDK API`或者`Webhook`的方式，设置自定义Condition。

### ReadinessGates

```yaml
kind: GameServer
...
spec:
  readinessGates:
    - network.ocgi.dev/lb-ready
status:
  conditions:
    - type: "network.ocgi.dev/lb-ready"
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
```

当Squad在变更时，需要保证所有`ReadinessGates Condition`都为`True`，`Carrier controller`才认为新的副本状态是Ready。

### DeletableGates

```yaml
kind: GameServer
...
spec:
  deletableGates:
    - carrier.ocgi.dev/has-no-player
status:
  conditions:
    - type: "carrier.ocgi.dev/has-no-player"
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
```

当Squad在变更，或者缩容时，需要保证`DeletableGates Condition`都为`True`，才能删除相应的副本。

## GameServer访问SDK Server

`SDK-Server`启动后，会监听一个gRPC和一个HTTP端口，`Carrier controller`会将端口信息作为环境变量写到`GameServer`容器：

```shell script
# kubectl exec -it squad-example-54b764fd57-2bkhx -c server /bin/bash
[root@squad-example-54b764fd57-2bkhx /]# env | grep SDK
CARRIER_SDK_GRPC_PORT=9020
CARRIER_SDK_HTTP_PORT=9021
```

* CARRIER_SDK_GRPC_PORT: gRPC服务端口，默认是9020
* CARRIER_SDK_HTTP_PORT: http服务端口，默认是9021

### SDK API

参考 [SDK gRPC API](https://github.com/ocgi/carrier-sdk/blob/master/docs/API.md) 和 [SDK HTTP API](https://github.com/ocgi/carrier-sdk/blob/master/docs/HTTP_API.md).

* [go sdk](https://github.com/ocgi/carrier-sdk/tree/master/sdks/sdkgo)

  [simple-tcp](https://github.com/ocgi/sdk-examples/tree/master/simple-tcp) is an example of using go sdk.

* [c++ sdk](https://github.com/ocgi/carrier-sdk/tree/master/sdks/sdkcpp)

  [sdk-examples-cpp](https://github.com/ocgi/sdk-examples-cpp) is an example of using c++ sdk.


## 自定义Webhook

如果应用程序不想调用`SDK API`，可以配置相应的Webhook，由`SDK-Server`来调用Webhook。

1.首先，我们定义一个Webhook服务。例如：

```yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: WebhookConfiguration
metadata:
  name: ds-webhook
  namespace: default
webhooks:
  - clientConfig:
      url: http://ds.carrier.dev/server-ready
    name: server-ready
    type: ReadinessWebhook
```

Webhook的详细定义参考[WebhookConfiguration](https://github.com/ocgi/carrier/blob/master/pkg/apis/carrier/v1alpha1/webhookconfiguration.go).

```go
// WebhookConfiguration is the data structure for a WebhookConfiguration resource.
type WebhookConfiguration struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Webhooks []Configurations `json:"webhooks"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// WebhookConfigurationList is a list of WebhookConfiguration resources
type WebhookConfigurationList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`

	Items []WebhookConfiguration `json:"items"`
}

type Configurations struct {
	ClientConfig   v1.WebhookClientConfig `json:"clientConfig"`
	Name           *string                `json:"name,omitempty"`
	Type           *string                `json:"type,omitempty"`
	TimeoutSeconds *int32                 `json:"timeoutSeconds,omitempty"`
	PeriodSeconds  *int32                 `json:"periodSeconds,omitempty"`
}
```

其中`Type`可以为
  - ReadinessWebhook

    对应`Configurations`中`Name`为 readinessGates中的值

  - DeletableWebhook

    对应`Configurations`中`Name`为 deletableGates中的值

  - ConstraintWebhook

    当ConstraintWebhook配置之后，sdk server会自动watch Gameserver的Constraint, 随后调用用户的webhook

2.配置`GameServer`访问Webhook。

```yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: GameServer
metadata:
  annotations:
    carrier.ocgi.dev/webhook-config-name: ds-webhook # should be the webhook name
  name: ds-server
  namespace: default
spec:
  health:
    disabled: true # 不检查GameServer的健康状况。
  readinessGates:
    - server-ready # readiness gate name should be same as the readiness gate name in webhook
```

3.Webhook Server

详细参考[webhook example](https://github.com/ocgi/carrier-sdk/tree/master/examples).

webhook server对应的API

```go

// WebhookRequest defines the request to webhook endpoint
type WebhookRequest struct {
	// UID is used for tracing the request and response.
	UID types.UID `json:"uid"`
	// Name is the name of the GameServer
	Name string `json:"name"`
	// Namespace is the workload namespace
	Namespace string `json:"namespace"`
	// Constraints describes the constraints of game server.
	// This filed may be added or changed by controller or manually.
	// If anyone of them is `NotInService` and Effective is `True`,
	// We should react to the constraint.
	Constraints []Constraint `json:"constraints,omitempty"`
}

// WebhookResponse defines the response of webhook server
type WebhookResponse struct {
	// UID is used for tracing the request and response.
	// It should be same as it in the request.
	UID types.UID `json:"uid"`
	// Set to false if should not allow
	Allowed bool `json:"allowed"`
	// Reason for this result.
	Reason string `json:"reason,omitempty"`
}

// Constraint describes the constraint info of game server.
type Constraint struct {
	// Type is the ConstraintType name, e.g. NotInService.
	Type string `json:"type"`
	// Effective describes whether the constraint is effective.
	Effective *bool `json:"effective,omitempty"`
	// Message explains why this constraint is added.
	Message string `json:"message,omitempty"`
	// TimeAdded describes when it is added.
	TimeAdded *time.Time `json:"timeAdded,omitempty"`
}

// WebhookReview is passed to the webhook with a populated Request value,
// and then returned with a populated Response.
type WebhookReview struct {
	Request  *WebhookRequest  `json:"request"`
	Response *WebhookResponse `json:"response"`
}
```

1. 开发的webhook server会收到`WebhookReview`, 但是只包含 `Request`, webhook server返回`WebhookReview`, 但是需要包含 `Response`;

2. 对于`ReadinessWebhook`、`DeletableWebhook`，收到的`Request`会包含`GameServer` Name、 Namespace, 
   用户的Response需要返回`Allowed`, `Reason`(可选);

3. 对于`ConstraintWebhook`, 收到的`Request`会包含`GameServer` Name、 Namespace、Constraint.
   用户的Response可以不返回任何有效信息，sdk-server当前未校验用户的返回信息。

## Reference

更多信息参考[carrier-sdk](https://github.com/ocgi/carrier-sdk)。
