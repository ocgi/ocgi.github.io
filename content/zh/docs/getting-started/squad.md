---
title: "创建Squad"
weight: 2
---

`Squad`代表一组游戏后端Server(`GameServer`)，它们具有相同的资源配置，并由`Carrier controller`维持该组GameServer在指定的副本数量。它控制该组GameServer的发布和更新。

## Create a Squad

创建一个2个副本的Squad:

```shell script
cat <<EOF | kubectl apply -f -
apiVersion: carrier.ocgi.dev/v1alpha1
kind: Squad
metadata:
  name: squad-example
  namespace: default
spec:
  replicas: 2
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        foo: bar
    spec:
      ports:
      - container: server
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
EOF
```

* 查看Squad信息

```shell script
# kubectl get sqd
NAME            SCHEDULING      DESIRED   CURRENT   UP-TO-DATE   READY   AGE
squad-example   MostAllocated   2         2         2            2       4s
```

其中：
- NAME 描述集群中Squad的名称
- SCHEDULING 调度策略。 默认为`MostAllocated`，支持`MostAllocated`、`LeastAllocated`
  - MostAllocated: 适用动态集群，会注入InterPodAffinity使得Pod尽量调度到一台机器，保证装箱率，同时缩容时影响尽量少的GameServer；
  - LeastAllocated: 原生的调度策略，调度到资源使用较少的节点，集群伸缩时影响的GameServer较多。  
- DESIRED 期望的副本数，与`.spec.replicas`描述的一致
- CURRENT 当前的副本数
- UP-TO-DATE 显示为了打到期望状态已经更新的副本数
- READY 已就绪的副本数
- AGE 显示应用程序运行的时间

查看GameServer和Pod信息：

```shell script
# kubectl get gs
NAME                             STATE     ADDRESS      PORT   PORTRANGE   NODE             AGE
squad-example-75ddb545d9-cqgph   Running   172.16.5.9                      172.16.212.247   21s
squad-example-75ddb545d9-nzvkx   Running   172.16.5.8                      172.16.212.247   21s

# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
squad-example-75ddb545d9-cqgph   2/2     Running   0          42s
squad-example-75ddb545d9-nzvkx   2/2     Running   0          42s
```

* 访问GameServer

```shell script
# telnet 172.16.5.8 7654
Trying 172.16.5.8...
Connected to 172.16.5.8.
Escape character is '^]'.
VERSION
Version: 0.1
```

## Scale the Squad

* Scale up

将`squad-example`的副本数量从2个扩容到4个：

```shell
# kubectl scale sqd squad-example --replicas=4
squad.carrier.ocgi.dev/squad-example scaled

# kubectl get gs
NAME                             STATE     ADDRESS      PORT   PORTRANGE   NODE             AGE
squad-example-75ddb545d9-66k6p   Running   172.16.5.1                      172.16.212.247   8s
squad-example-75ddb545d9-cqgph   Running   172.16.5.9                      172.16.212.247   90m
squad-example-75ddb545d9-fm228   Running   172.16.5.0                      172.16.212.247   8s
squad-example-75ddb545d9-nzvkx   Running   172.16.5.8                      172.16.212.247   90m

# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
squad-example-75ddb545d9-66k6p   2/2     Running   0          16s
squad-example-75ddb545d9-cqgph   2/2     Running   0          90m
squad-example-75ddb545d9-fm228   2/2     Running   0          16s
squad-example-75ddb545d9-nzvkx   2/2     Running   0          90m
```

* Scale down

将`squad-example`的副本数量从4个扩容到3个：

```shell
# kubectl scale sqd squad-example --replicas=3
squad.carrier.ocgi.dev/squad-example scaled

# kubectl get gs  
NAME                             STATE     ADDRESS      PORT   PORTRANGE   NODE             AGE
squad-example-75ddb545d9-66k6p   Running   172.16.5.1                      172.16.212.247   2m22s
squad-example-75ddb545d9-fm228   Running   172.16.5.0                      172.16.212.247   2m22s
squad-example-75ddb545d9-nzvkx   Running   172.16.5.8                      172.16.212.247   92m

# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
squad-example-75ddb545d9-66k6p   2/2     Running   0          2m37s
squad-example-75ddb545d9-fm228   2/2     Running   0          2m37s
squad-example-75ddb545d9-nzvkx   2/2     Running   0          92m
```

## Update the Squad

更新`squad-example`的镜像，改成`ocgi/simple-tcp:0.2`:

```shell
# kubectl patch squad squad-example --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/template/spec/containers/0/image", "value":"ocgi/simple-tcp:0.2"}]'
# kubectl get gs  
NAME                             STATE     ADDRESS      PORT   PORTRANGE   NODE             AGE
squad-example-5df4fb94c6-75565   Running   172.16.5.4                      172.16.212.247   34s
squad-example-5df4fb94c6-lzb92   Running   172.16.5.2                      172.16.212.247   43s
squad-example-5df4fb94c6-sjv9f   Running   172.16.5.3                      172.16.212.247   43s

# telnet 172.16.5.2 7654
Trying 172.16.5.2...
Connected to 172.16.5.2.
Escape character is '^]'.
VERSION
Version: 0.2
```

可以看到，`squad-example`的镜像已经更新到`ocgi/simple-tcp:0.2`。

## 回滚操作

> 有时在我们发布镜像后，发现线上存在BUG，或者不稳定，希望回退到之前的版本，那么就可以通过squad的回退功能来实现。在squad中保存着一定数量（默认10个）版本，以便可以随时回滚指定的版本

1. 例如我们在更新镜像`ocgi/simple-tcp:0.2`后发现BUG，希望回退到上一个版本`ocgi/simple-tcp:0.1`，我们可以执行。其中`revision`设置为0，表示回退到最近一个版本。

```shell
# kubectl patch squad squad-example --type='json' -p='[{"op": "add", "path": "/spec/rollbackTo", "value": {"revision": 0}}]'
```

2. 查看回退后Squad的镜像，是否符合预期，执行

```shell script
# kubectl get sqd squad-example -o  jsonpath="{..image}"
ocgi/simple-tcp:0.1

# kubectl get gs  
NAME                             STATE     ADDRESS      PORT   PORTRANGE   NODE             AGE
squad-example-75ddb545d9-6n7jh   Running   172.16.5.6                      172.16.212.247   24s
squad-example-75ddb545d9-l4dpc   Running   172.16.5.7                      172.16.212.247   19s
squad-example-75ddb545d9-lflnv   Running   172.16.5.5                      172.16.212.247   24s

# telnet 172.16.5.5 7654
Trying 172.16.5.5...
Connected to 172.16.5.5.
Escape character is '^]'.
VERSION
Version: 0.1
```

可见这里的镜像已回退成`ocgi/simple-tcp:0.1`。

## Reference

关于Squad更多信息请参考[Squad介绍](/zh/docs/guides/squad_details).
