---
title: "设置GamerServer缩容的优先级"
weight: 5
---

## 背景

一般来说，每个游戏服务器上的玩家数量会不同。当我们缩容时，可以选择用户玩家少的副本，进行删除。这样，可以让应用侧的缩容开销更小，同时，也可以提高底层资源的利用效率。

我们可以根据一些业务侧指标，给每个游戏服务器设置一定的优先级。缩容时，我们选择优先级低的副本删除。

## 核心方案 

1. 给GameServer增加carrier.ocgi.dev/gs-deletion-cost(int64) annotation， annotation可以由用户调用sdk加上，或者由我们的组件cost-server加上。
   - cost-server根据squad里指定的 carrier.ocgi.dev/gs-cost-metrics-name annotation 从metric server获取数据， 给gs加上carrier.ocgi.dev/gs-deletion-cost annotation。
   - 如果GameServer未设置carrier.ocgi.dev/gs-deletion-cost 按照Int64 Max计算。
   - 如果所有GameServer都未设置，进入原先的逻辑，优先选择节点上GameServer较少的节点上的进行缩容，或者按照创建时间最老进行缩容。

2. carrier controller，按照该值从小到大顺序排序， 缩容掉top N。

## cost-server 简介

1. `cost-server`watch `Squad`的创建和更新；
2. `cost-server`从名为`carrier.ocgi.dev/gs-cost-metrics-name`的annotation获取对应的metric名；
3. 如果metric是`cpu` 或者 `memory`，`cost-server`将从`metric server`获取数据；如果是其余metric将从`custom metric server`获取数据；
4. 最终的数据将会写到GameServer的annotation上，例如：`carrier.ocgi. dev/gs-deletion-cost: "2101248000"`；

## 使用方式

1. 如果是使用`cpu`、`memory`等指标，用户用只需要在Squad上指定`carrier.ocgi.dev/gs-cost-metrics-name`；
2. 如果用户需要使用自定义指标，如`连接数`等，用户可以采用以下2种方式之一：
- 给Squad加上`carrier.ocgi.dev/gs-cost-metrics-name: 连接数`, 该方式存在一定延迟
- 借助SDK的SetAnnotation方法, 给每个GameServer设置`carrier.ocgi.dev/gs-deletion-cost`

## 示例

使用cost-server自动给GameServer加`cost`

### 创建 squad

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

### 查看 GameServers

```shell
# kubectl get gs -o wide
NAME                      STATE     ADDRESS         PORT   EXTERNALIP   PORTRANGE     NODE            AGE
squad2-56fbd4d75d-4js2f   Running   192.168.113.169                                   192.168.0.115   64m
squad2-56fbd4d75d-cqxtz   Running   192.168.113.165                                   192.168.0.115   64m
squad2-56fbd4d75d-gsh76   Running   192.168.113.166                                   192.168.0.129   64m
squad2-56fbd4d75d-mmcgf   Running   192.168.113.168                                   192.168.0.213   64m
```

### 查看 Cost

```shell
# kubectl get gs -o custom-columns="Name":".metadata.name","Cost":".metadata.annotations.carrier\.ocgi\.dev/gs-deletion-cost","UpdateTime":".metadata.annotations.carrier\.ocgi\.dev/deletion-cost-update-time"
Name                      Cost         UpdateTime
squad2-56fbd4d75d-4js2f   2101248000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-cqxtz   2191360000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-gsh76   2007040000   2021-04-15 08:20:58 +0000 UTC
squad2-56fbd4d75d-mmcgf   2068480000   2021-04-15 08:20:58 +0000 UTC
```

### 缩容

```shell
kubectl scale sqd squad2 --replicas=3
```

### 再次查看Cost

```shell
# kubectl get gs -o custom-columns="Name":".metadata.name","Cost":".metadata.annotations.carrier\.ocgi\.dev/gs-deletion-cost","UpdateTime":".metadata.annotations.carrier\.ocgi\.dev/deletion-cost-update-time"
Name                      Cost         UpdateTime
squad2-56fbd4d75d-4js2f   2101248000   2021-04-15 08:21:43 +0000 UTC
squad2-56fbd4d75d-cqxtz   2191360000   2021-04-15 08:21:43 +0000 UTC
squad2-56fbd4d75d-mmcgf   2068480000   2021-04-15 08:21:43 +0000 UTC
```

可以看到，最小`Cost`的副本被删除。