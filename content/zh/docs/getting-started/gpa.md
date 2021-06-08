---
title: "Create a GeneralPodAutoscaler"
weight: 3
---

`GeneralPodAutoscaler(GPA)`完全兼容[K8s HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)的功能。同时，GPA支持Crontab、Webhook等方式。

## 前置条件

创建一个如下的squad

```shell script
# cat <<EOF | kubectl apply -f -
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
        foo: squad-example
    spec:
      ports:
      - container: simple-udp
        containerPort: 7654
        hostPort: 7777
        name: default
        portPolicy: Static
        protocol: UDP
      template:
        spec:
          containers:
          - image: ocgi/simple-tcp:0.1
            imagePullPolicy: Always
            name: server
          serviceAccount: carrier-sdk
          serviceAccountName: carrier-sdk
EOF
```

## 定时扩缩模式

基于Crontab语法，支持多时间段

```shell script
# cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.ocgi.dev/v1alpha1
kind: GeneralPodAutoscaler
metadata:
  name: pa-test1
spec:
  maxReplicas: 8
  minReplicas: 2
  scaleTargetRef:
    apiVersion: carrier.ocgi.dev/v1alpha1
    kind: Squad
    name: squad-example
  time:
    ranges:
    - desiredReplicas: 4
      schedule: '*/1 2-3 * * *'
    - desiredReplicas: 6
      schedule: '*/1 4-5 * * *'
EOF

# kubectl get pa pa-squad
NAME       MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad   1             8             4         4         Squad        squad-example
# date
Wed Nov 25 11:58:28 CST 2020
```


## Webhook

webhook模式，支持业务自定义开发一个webhook，来定义何时伸缩，以下是一个例子：

保证有2个可用的空闲GameServer

```shell script
# cat <<EOF | kubectl apply -f -
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
EOF

# kubectl get pa pa-squad
NAME       MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad   1             8             2         4         Squad        squad-example
```

## Mix webhook and crontab

混合多种模式进行自动伸缩

```shell script
# cat <<EOF | kubectl apply -f -
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
  time:
    ranges:
    - desiredReplicas: 4
      schedule: '*/1 10-23 * * *'
  webhook:
    parameters:
      buffer: "2"
    service:
      name: gpa-webhook
      namespace: kube-system
      path: scale
      port: 8000
EOF

# kubectl get pa pa-squad
NAME       MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad   1             8             2         4         Squad        squad-example
```

## Metric

通过metric的方式弹性伸缩

### In-tree metrics
内置的指标包括： cpu、memory

```shell script
# cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.ocgi.dev/v1alpha1
kind: GeneralPodAutoscaler
metadata:
  name: pa-squad-metric
spec:
  maxReplicas: 10
  minReplicas: 2
  metric:
    metrics:
    - resource:
        name: cpu
        target:
          averageValue: 20
          type: AverageValue
      type: Resource
    - resource:
        name: memory
        target:
          averageValue: 50m
          type: AverageValue
      type: Resource
  scaleTargetRef:
    apiVersion: carrier.ocgi.dev/v1alpha1
    kind: Squad
    name: squad-example1
EOF

# kubectl get pa pa-squad-metric
NAME              MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad-metric   2             10            4         2         Squad        squad-example1

# kubectl get pa pa-squad-metric
NAME              MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad-metric   2             10            4         8         Squad        squad-example1

# kubectl top pod
NAME                                     CPU(cores)   MEMORY(bytes)              
squad-example1-8665fc7ff5-bdvcj          1m           9Mi             
squad-example1-8665fc7ff5-x7znq          1m           10Mi            
squad-example1-8665fc7ff5-xrkng          5m           10Mi            
squad-example1-8665fc7ff5-xzntk          5m           10Mi            

# kubectl get pa pa-squad-metric
NAME              MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad-metric   2             10            10        10        Squad        squad-example1

# kubectl top pod
NAME                                     CPU(cores)   MEMORY(bytes)  
squad-example1-8665fc7ff5-8h5rs          1m           10Mi            
squad-example1-8665fc7ff5-bdvcj          1m           10Mi            
squad-example1-8665fc7ff5-kf4tz          1m           10Mi            
squad-example1-8665fc7ff5-kx5px          1m           10Mi            
squad-example1-8665fc7ff5-ldcm7          1m           8Mi             
squad-example1-8665fc7ff5-mknnk          1m           9Mi             
squad-example1-8665fc7ff5-wdlrl          1m           10Mi            
squad-example1-8665fc7ff5-x7znq          1m           10Mi            
squad-example1-8665fc7ff5-xrkng          1m           10Mi            
squad-example1-8665fc7ff5-xzntk          1m           10Mi  
```

### custom metric

自定义指标：我们提供了出cpu 内存之外的指标，可以使用网卡流量等，或者业务自定的一些指标


```shell script
# cat <<EOF | kubectl apply -f -
apiVersion: autoscaling.ocgi.dev/v1alpha1
kind: GeneralPodAutoscaler
metadata:
  name: pa-squad-metric-custom
spec:
  maxReplicas: 10
  minReplicas: 2
  metric:
    metrics:
      - type: Pods
        pods:
          metric:
            name: memory_rss
          target:
            averageValue: 10m
            type: AverageValue
  scaleTargetRef:
    apiVersion: carrier.ocgi.dev/v1alpha1
    kind: Squad
    name: squad-example2
EOF

# kubectl get pa pa-squad-metric-custom
NAME                     MINREPLICAS   MAXREPLICAS   DESIRED   CURRENT   TARGETKIND   TARGETNAME
pa-squad-metric-custom   2             10            10        10        Squad        squad-example2
```

## Reference

关于GPA更多信息请参考[GeneralPodAutoscaler介绍](/zh/docs/guides/gpa).
