---
title: "GamerServer scaling down priority"
weight: 5
---

## Background

Generally speaking, the number of players on each game server will be different. When we scaling down, we can select a replica with fewer users and delete it. In this way, the scaling-down overhead on the application side can be smaller, and at the same time, the utilization efficiency of the underlying resources can be improved.

We can set a certain priority for each game server based on some application metrics. When scaling down, we choose the replica with the lower priority to delete.

## Implementation

1. Add `carrier.ocgi.dev/gs-deletion-cost(int64)` annotation to GameServer. The annotation can be added by user calling SDK or added by our component `cost-server`.
  - The cost-server obtains data from the metric server according to the `carrier.ocgi.dev/gs-cost-metrics-name` annotation specified in the squad, and adds `carrier.ocgi.dev/gs-deletion-cost` annotation to the gs.
  - If the GameServer does not set `carrier.ocgi.dev/gs-deletion-cost`, it is calculated according to Int64 Max.
  - If all GameServers are not set, enter the original logic, and select the node with fewer GameServers on the node for shrinking, or shrinking according to the oldest creation time.

2. The Carrier controller is sorted in ascending order of the value, and the top N is reduced.

## Cost-server Introduction

1. cost-server watch the create and update event of `Squad`;
2. The cost-server obtains the corresponding metric name from the annotation named `carrier.ocgi.dev/gs-cost-metrics-name`;
3. If the metric is cpu or memory, the cost-server will obtain data from the metric server; if it is other metrics, it will obtain data from the custom metric server;
4. The final data will be written to the annotation of GameServer, for example:` carrier.ocgi. dev/gs-deletion-cost: "2101248000"`;

## Usage

1. If you are using metrics such as cpu and memory, only need to specify `carrier.ocgi.dev/gs-cost-metrics-name` on `Squad`;
2. If you want to use custom metrics, such as the number of connections, the user can use one of the following two methods:
  - Add `carrier.ocgi.dev/gs-cost-metrics-name: <number of connections>` to `Squad`, there is a certain delay in this method
  - With the SDK SetAnnotation method, set `carrier.ocgi.dev/gs-deletion-cost` for each GameServer

## Examples

Set cost for GameServer based on the cost-server.

### Create a Squad

```shell
# cat <<EOF | kubectl apply -f -
apiVersion: carrier.ocgi.dev/v1alpha1
kind: Squad
metadata:
  annotations:
    carrier.ocgi.dev/gs-cost-metrics-name: memory
    network.cni/networkname: tke-direct-eni
  name: squad2
spec:
  replicas: 4
  scheduling: MostAllocated
  selector:
    matchLabels:
      carrier.ocgi.dev/squad: squad2
  template:
    metadata:
      annotations:
        network.cni/networkname: tke-direct-eni
      labels:
        foo: bar
    spec:
      health:
        disabled: true
      ports:
        - container: simple-tcp
          containerPort: 7654
          name: default
          protocol: TCP
      sdkServer:
        grpcPort: 9020
        httpPort: 9021
        logLevel: Info
      template:
        spec:
          containers:
            - image: ocgi/simple-tcp:0.1
              name: server
          serviceAccountName: carrier-sdk
```

### Fetch GameServers

```shell
# kubectl get gs -o wide
NAME                      STATE     ADDRESS         PORT   EXTERNALIP   PORTRANGE     NODE            AGE
squad2-56fbd4d75d-4js2f   Running   192.168.113.169                                   192.168.0.115   64m
squad2-56fbd4d75d-cqxtz   Running   192.168.113.165                                   192.168.0.115   64m
squad2-56fbd4d75d-gsh76   Running   192.168.113.166                                   192.168.0.129   64m
squad2-56fbd4d75d-mmcgf   Running   192.168.113.168                                   192.168.0.213   64m
```

### Check the Cost

```shell
# kubectl get gs -o custom-columns="Name":".metadata.name","Cost":".metadata.annotations.carrier\.ocgi\.dev/gs-deletion-cost","UpdateTime":".metadata.annotations.carrier\.ocgi\.dev/deletion-cost-update-time"
Name                      Cost         UpdateTime
squad2-56fbd4d75d-4js2f   2101248000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-cqxtz   2191360000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-gsh76   2007040000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-mmcgf   2068480000   2021-04-15 08:20:58 +0000 UTC
```

### Scale-down the Squad

```shell
kubectl scale sqd squad2 --replicas=3
```

### Check the Cost

```shell
# kubectl get gs -o custom-columns="Name":".metadata.name","Cost":".metadata.annotations.carrier\.ocgi\.dev/gs-deletion-cost","UpdateTime":".metadata.annotations.carrier\.ocgi\.dev/deletion-cost-update-time"
Name                      Cost         UpdateTime
squad2-56fbd4d75d-4js2f   2101248000   2021-04-15 08:21:43 +0000 UTC
squad2-56fbd4d75d-cqxtz   2191360000   2021-04-15 08:21:43 +0000 UTC
squad2-56fbd4d75d-mmcgf   2068480000   2021-04-15 08:21:43 +0000 UTC
```

As we can see, controller deleted the replica with the minimal Cost.