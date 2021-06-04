---
title: "Create a GameServer"
weight: 1
---

`GameServer` represents a single game server. It is based on [`K8s Pod`](https://kubernetes.io/docs/concepts/workloads/pods/) implementation and is a further abstraction of K8s Pod.

## Create a GameServer

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

## Fetch the GameServer Status

```shell script
# kubectl get gs  
NAME                      STATE     ADDRESS         PORT   EXTERNALIP   PORTRANGE   NODE            AGE
simple-tcp-example        Running   172.16.5.3                                      172.16.18.47    16s

# kubectl get pods
NAME                           READY   STATUS              RESTARTS   AGE
simple-tcp-example             2/2     Running             0          48s
```

Each field of output represents:
- STATE describes the state of GameServer, including the following states
  - Starting: means the pod phase of GameServer is Pending or pod not existing
  - Running: means the pod phase of GameServer is Running
  - Exited: means GameServer has exited
  - Failed: means the pod phase of GameServer is Failed
  - Unknown: means the pod phase of GameServer is Unkown
  - ADDRESS corresponds to the ip of the Pod
- PORT The port used by the application to provide services
- EXTERNALIP provides access IP, such as LB address
- PORTRANGE The port range where the application provides services
- Node node to which the application belongs
- AGE shows the running time of the application
- LABELS The label of the application. The label prefixed with `carrier.ocgi.dev` is created by the system by default

Details of GameServer:

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

The `GameServer.Address` is the IP address of the `GameServer` container.

## Access the GameServer

```shell
# telnet 172.16.5.3 7654
Trying 172.16.5.3...
Connected to 172.16.5.3.
Escape character is '^]'.
FILLED TRUE
ACK: FILLED
...
```

The `GameServer` condition will be changed:

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

For more information, please refer to the [Simple TCP Server](https://github.com/ocgi/sdk-examples/tree/master/simple-tcp).
