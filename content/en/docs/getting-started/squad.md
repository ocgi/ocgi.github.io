---
title: "Create a Squad"
weight: 2
---

`Squad` represents a group of game servers (`GameServer`), they have the same resource configuration, 
and the Carrier controller maintains the specified number of copies of the group of GameServer. 
It controls the update and scaling of this group of GameServer.

## Create a Squad

Create a Squad with 2 replicas:

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
      template:
        spec:
          containers:
          - image: ocgi/simple-tcp:0.1
            name: server
EOF
```

* Fetch the Squad status

```shell script
# kubectl get sqd
NAME            SCHEDULING      DESIRED   CURRENT   UP-TO-DATE   READY   AGE
squad-example   MostAllocated   2         2         2            2       4s
```
Each field of GameServer represents:
- NAME describes the name of the Squad in the cluster
- SCHEDULING scheduling strategy. The default is `MostAllocated`, support `MostAllocated`, `LeastAllocated`
   - MostAllocated: Applicable to dynamic clusters, InterPodAffinity will be injected to make Pod dispatch to one machine as much as possible to ensure the packing rate, and at the same time, the GameServer that has the least impact when shrinking;
   - LeastAllocated: Native scheduling strategy, scheduling to nodes with less resource usage, more GameServers will be affected when the cluster is scaled.
- DESIRED The number of replicas expected, consistent with the description of `.spec.replicas`
- CURRENT current number of copies
- UP-TO-DATE shows the number of copies that have been updated in order to reach the desired status
- READY The number of copies that are ready
- AGE shows the running time of the application

View GameServer and Pod information:
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

* Access the GameServer

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

Scale up the number of replicas of `squad-example` from 2 to 4:

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

Scale down the number of replicas of `squad-example` from 4 to 3:

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

Update the `squad-example` image to `ocgi/simple-tcp:0.2`:

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

As we can see, the image of `squad-example` has been updated to `ocgi/simple-tcp:0.2`.

## Rollback update

Sometimes after we complete the update, we find that there is a BUG or unstable online, and we want to roll back to the previous version, then we can achieve it through the Squad rollback function. 
A certain number of versions (10 by default) are saved in Squad so that the specified version can be rolled back at any time.

For example, we found a BUG after updating to `ocgi/simple-tcp:0.2`, and hope to roll back to the previous version `ocgi/simple-tcp:0.1`, we can execute it.
The revision is set to 0, which means to roll back to the latest version.

```shell script
# kubectl patch squad squad-example --type='json' -p='[{"op": "add", "path": "/spec/rollbackTo", "value": {"revision": 0}}]'
```

Check the `squad-example` image:

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

As we can see, the image has been rollbacked to `ocgi/simple-tcp:0.1`.

## Reference

For more information, please refer to the [Squad introduction](/en/docs/guides/squad_details).
