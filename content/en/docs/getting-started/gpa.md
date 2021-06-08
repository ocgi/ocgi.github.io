---
title: "Create a GeneralPodAutoscaler"
weight: 3
---

[`GeneralPodAutoscaler (GPA)`](https://github.com/ocgi/general-pod-autoscaler) is fully compatible with the functions of [K8s HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). At the same time, GPA supports `Crontab`, `Webhook` and other modes.

## Pre-requirement

Create a squad.

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

## Crontab mode

Based on Crontab format, support multiple time periods.

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


## Webhook mode

The `Webhook` mode supports the application provide a webhook to define when to scale. The following is an example:

Ensure that there are 2 free GameServers available.

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

We can mix `Webhook` and `Crontab` mode at the same time.

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

## Metrics

Auto-scaling based on the metrics.

### In-tree metrics

In-tree metrics include `cpu` and `memory`.

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

### custom metrics

GPA support custom metrics other than cpu memory, you can use network card traffic, etc., or some metrics defined by the application.

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

More information about GPA refer to [GeneralPodAutoscaler introduction](/en/docs/guides/gpa).
