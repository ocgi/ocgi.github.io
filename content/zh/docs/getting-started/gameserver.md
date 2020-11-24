---
title: "创建GameServer"
weight: 1
---

`GameServer`代表单个游戏后端Server。它基于[`K8s Pod`](https://kubernetes.io/docs/concepts/workloads/pods/)实现，是对`K8s Pod`的进一步抽象。

## 创建GameServer

```shell script
cat <<EOF | kubectl apply -f -
apiVersion: "carrier.ocgi.dev/v1alpha1"
kind: GameServer
metadata:
  name: "simple-tcp-example"
spec:
  ports:
  - name: default
    containerPort: 7654
    protocol: TCP
  health:
    disabled: true
  template:
    spec:
      containers:
      - name: server
        image: ocgi/simple-tcp:0.1
        resources:
          requests:
            memory: "32Mi"
            cpu: "20m"
          limits:
            memory: "32Mi"
            cpu: "20m"
      serviceAccount: carrier-sdk
      serviceAccountName: carrier-sdk
EOF
```

## 查看GameServer信息

```shell script
# kubectl get gs  
NAME                      STATE     ADDRESS         PORT   EXTERNALIP   PORTRANGE   NODE            AGE
simple-tcp-example        Running   172.16.5.3                                      172.16.18.47    16s

# kubectl get pods
NAME                           READY   STATUS              RESTARTS   AGE
simple-tcp-example             2/2     Running             0          48s
```

各个字段的含义：
- STATE 描述GameServer的状态，包括以下几种状态
  - Starting: means the pod phase of GameServer is Pending or pod not existing
  - Running: means the pod phase of GameServer is Running
  - Exited: means GameServer has exited
  - Failed: means the pod phase of GameServer is Failed
  - Unknown: means the pod phase of GameServer is Unkown

- ADDRESS 对应Pod的ip
- PORT 应用程序提供服务的端口
- EXTERNALIP 提供访问的IP，例如LB地址
- PORTRANGE 应用程序提供服务的端口段
- NODE 应用程序所属的node节点
- AGE 显示应用程序运行的时间
- LABELS 应用程序的标签。`carrier.ocgi.dev`前缀的标签为系统默认创建

GameServer的详细信息：

```shell script
# kubectl describe gs/simple-tcp-example
Name:         simple-tcp-example
Namespace:    default
Labels:       <none>
Annotations:  carrier.ocgi.dev/container: docker://6ff5c9538916b7356347662a1b8c7818f42ddfbd663c4838f8d8ddea8aa51e01
              kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"carrier.ocgi.dev/v1alpha1","kind":"GameServer","metadata":{"annotations":{},"name":"simple-tcp-example","namespace":"defaul...
API Version:  carrier.ocgi.dev/v1alpha1
Kind:         GameServer
Metadata:
  Creation Timestamp:  2021-05-17T07:05:24Z
  Generation:          2
  Resource Version:    341523019
  Self Link:           /apis/carrier.ocgi.dev/v1alpha1/namespaces/default/gameservers/simple-tcp-example
  UID:                 40821b17-b6de-11eb-8908-525400034523
Spec:
  Health:
    Disabled:  true
  Ports:
    Container Port:  7654
    Name:            default
    Protocol:        TCP
  Sdk Server:
  Template:
    Metadata:
      Creation Timestamp:  <nil>
    Spec:
      Containers:
        Image:  ocgi/simple-tcp:0.1
        Name:   server
        Resources:
          Limits:
            Cpu:     20m
            Memory:  32Mi
          Requests:
            Cpu:             20m
            Memory:          32Mi
      Service Account:       carrier-sdk
      Service Account Name:  carrier-sdk
Status:
  Address:  172.16.5.3
  Conditions:
    Last Probe Time:       2021-05-17T07:05:36Z
    Last Transition Time:  2021-05-17T07:05:36Z
    Status:                True
    Type:                  carrier.dev/ready
  Node Name:               172.16.18.47
  State:                   Running
Events:
  Type    Reason    Age                From                   Message
  ----    ------    ----               ----                   -------
  Normal            80s                gameserver-controller  Creating pod simple-tcp-example
  Normal  Starting  80s                gameserver-controller  Pod simple-tcp-example controlled by GameServer created
  Normal  Running   68s                gameserver-controller  Address and port populated
  Normal  Running   68s (x3 over 68s)  gameserver-controller  Waiting for receiving ready message
  Normal  Running   68s                carrier-sdkserver      Condition changed: carrier.dev/ready to True
```

其中,`GameServer.Address`为`game server`容器的IP地址。

## 访问GameServer

```shell
# telnet 172.16.5.3 7654
Trying 172.16.5.3...
Connected to 172.16.5.3.
Escape character is '^]'.
FILLED TRUE
ACK: FILLED
...
```

这会设置`GameServer`的Condition:


```shell
# kubectl get gs/simple-tcp-example -o yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: GameServer
...
status:
  address: 172.16.5.3
  conditions:
  - lastProbeTime: "2021-05-17T07:07:37Z"
    lastTransitionTime: "2021-05-17T07:07:37Z"
    status: "True"
    type: carrier.dev/filled
  - lastProbeTime: "2021-05-17T07:07:46Z"
    lastTransitionTime: "2021-05-17T07:05:36Z"
    status: "True"
    type: carrier.dev/ready
  nodeName: 172.16.18.47
  state: Running
```

## Reference

更多参考[Simple TCP Server](https://github.com/ocgi/sdk-examples/tree/master/simple-tcp).
