---
title: "Squad介绍"
weight: 1
---

## Squad

一个Squad控制器为GameServers和GameServerSets提供声明式的更新管理能力。用户可以通过描述Squad中的目标状态(例如副本数)，Squad控制器会按照一定的规则更改实际状态，最终使其达到期望的状态。

### 主要特点

- 管理一组GameServer
- 支持滚动更新发布
- 支持分批（灰度）发布
- 支持原地更新发布
- 支持资源自动弹性伸缩

## Squad更新策略

Squad通过`.spec.strategy`来指定更新GameServer的策略。`.spec.strategy.type`可以是 `Recreate`、`RollingUpdate`、`CanaryUpdate`、或者`InplaceUpdate`，默认为`RollingUpdate`

### Recreate策略

在创建GameServer前，所有现有的GameServer会被删除掉

### RollingUpdate策略

在指定`.spec.strategy.type`为RollingUpdate时，还可以指定`maxUnavailable` 和 `maxSurge` 来控制滚动更过程
- maxUnavailable：最大不可用GameServer数。是一个可选字段，用来指定更新过程中不可用的  GameServer 的个数上限。该值可以是绝对数字（例如，5），也可以是所需 GameServer 的百分比（例如，10%）。百分比值会转换成绝对数并去除小数部分。 如果 `.spec.strategy.rollingUpdate.maxSurge` 为 0，则此值不能为 0。 默认值为 25%
- maxSurge: 最大GameServer创建数。是一个可选字段，用来指定可以创建的超出期望  GameServer 个数的 GameServer 数量。此值可以是绝对数（例如，5）或所需 GameServer 的百分比（例如，10%）。 如果 MaxUnavailable 为 0，则此值不能为 0。百分比值会通过向上取整转换为绝对数。 此字段的默认值为 25%。

	> 例如：当此值为25%时，启动滚动更新后，会立即对新的 GameServerSet 扩容，同时保证新旧 GameServer 的总数不超过所需 GameServer 总数的 125%。一旦旧 GameServer 被杀死，新的 GameServerSet 可以进一步扩容， 同时确保更新期间的任何时候运行中的 GameServer 总数最多为所需 GameServer 总数的 125%

使用示例：

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

### CanaryUpdate策略

分批灰度更新策略，在CanaryUpdate策略下，还可以指定`type`和`threshold`来控制灰度过程
- type: 灰度过程中GameServer的更新规则，包括“deleteFirst”、“createFirst”或者“inplace”
  - deleteFirst：在创建新的GameServer时，先删除旧的GameServer
  - createFirst: 先创建新的GameServer，并且在新的GameServer Ready后，才会删除旧的GameServer
- threshold: 灰度的阀值，用于控制每批灰度的GameServer数，这里可以是绝对数字（例如，5），也可以是所有 GameServer 的百分比（例如，10%）

使用示例：

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

### InplaceUpdate策略

原地更新策略，以Update GameServer的方式来更新容器，而非重建整个GameServer，可以保持GameServer的IP不变，同时GameServer的挂载的内容（例如共享内存）得以保持下来。更新过程中通过`threshold`来控制更新的副本数
- threshold: 灰度的阀值，用于控制每批灰度的GameServer数，这里可以是绝对数字（例如，5），也可以是所有 GameServer 的百分比（例如，10%）

  > 更新时需要手动调整threshold值， 当所有的副本数都更新完之后，threshold会被重置成0,以防止下层更新时误操作 。

使用示例：

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