---
title: "Squad Introduction"
weight: 1
---

## Squad

A `Squad` controller provides declarative update management capabilities for GameServers and GameServerSets. The user can describe the target state in Squad (such as the number of replicas), and the Squad controller will change the actual state according to certain rules, and finally make it reach the desired state.

## main feature

- Manage a group of GameServer
- Support rolling update
- Support batch (gray) update
- Support in-place update
- Support GameServer auto-scaling

## Squad update strategy

Squad specifies the strategy for updating GameServer through `.spec.strategy`. `.spec.strategy.type` can be `Recreate`, `RollingUpdate` , `CanaryUpdate` or `InplaceUpdate`, the default is `RollingUpdate`.

### Recreate strategy

Before creating GameServer, all existing GameServer will be deleted.

## RollingUpdate strategy

When specifying `.spec.strategy.type` as `RollingUpdate`, you can also specify `maxUnavailable` and `maxSurge` to control the rolling update process.

- maxUnavailable

  The maximum number of unavailable GameServers. It is an optional field to specify the upper limit of the number of GameServers that are not available during the update process. The value can be an absolute number (for example, 5) or a percentage of the desired GameServer (for example, 10%). The percentage value is converted to an absolute number and the decimal part is removed. If `.spec.strategy.rollingUpdate.maxSurge` is 0, this value cannot be 0. The default value is 25%.

- maxSurge

  The maximum number of GameServer creations. It is an optional field to specify the number of GameServers that can be created beyond the expected number of GameServers. This value can be an absolute number (for example, 5) or a percentage of the desired GameServer (for example, 10%). If `MaxUnavailable` is 0, this value cannot be 0. The percentage value is converted to an absolute number by rounding up. The default value for this field is 25%.

  For example: when this value is 25%, after starting the rolling update, the new GameServerSet will be expanded immediately, while ensuring that the total number of old and new GameServers does not exceed 125% of the total number of required GameServers. Once the old GameServer is killed, the new GameServerSet can be further expanded, while ensuring that the total number of GameServers running at any time during the update period is at most 125% of the total number of GameServers required.


Example:

```yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: Squad
metadata:
  name: squad5
  namespace: default
spec:
  replicas: 1
  scheduling: MostAllocated
  selector:
    matchLabels:
      carrier.ocgi.dev/squad: squad5
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
...
```

### CanaryUpdate strategy

Batch gray-scale update strategy, under the `CanaryUpdate` strategy, you can also specify `type` and `threshold` to control the gray-scale process.

- type: GameServer update rules during the grayscale process, including "deleteFirst", "createFirst" or "inplace"
  - deleteFirst: When creating a new GameServer, delete the old GameServer first
  - createFirst: Create a new GameServer first, and delete the old GameServer only after the new GameServer is Ready
- threshold: The threshold of the gray level, used to control the number of GameServers in each batch of gray levels. It can be an absolute number (for example, 5) or a percentage of all GameServers (for example, 10%)

Example:

```yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: Squad
metadata:
  name: squad5
  namespace: default
spec:
  replicas: 1
  scheduling: MostAllocated
  selector:
    matchLabels:
      carrier.ocgi.dev/squad: squad5
  strategy:
    canaryUpdate:
      tpye: createFirst
      threshold: 10%
    type: CanaryUpdate
...
```

### InplaceUpdate strategy

In-place update strategy, update the container in the way of Update GameServer instead of rebuilding the entire GameServer, you can keep the IP of GameServer unchanged, and at the same time the content (such as shared memory) of GameServer can be maintained. Threshold is used to control the number of updated replicas during the update process.

- threshold: The threshold of the gray level, used to control the number of GameServers in each batch of gray levels. It can be an absolute number (for example, 5) or a percentage of all GameServers (for example, 10%)

  > It is necessary to manually adjust the `threshold` value when updating. After all the replicas are updated, the `threshold` will be reset to 0 to prevent misoperation during the update of the lower layer.

Example:

```yaml
apiVersion: carrier.ocgi.dev/v1alpha1
kind: Squad
metadata:
  name: squad5
  namespace: default
spec:
  replicas: 1
  scheduling: MostAllocated
  selector:
    matchLabels:
      carrier.ocgi.dev/squad: squad5
  strategy:
    inplaceUpdate:
      threshold: 1
    type: InplaceUpdate
...
```