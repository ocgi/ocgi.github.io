---
title: "GameServer自动伸缩流程"
weight: 4
---

`GameServer`的autoscaling的详细流程如下：

![autoscaler archtecture](/images/autoscaler_archtecture.jpg)

* `Autoscaler controller`根据GameServer的metric信息，计算Squad的合理副本数量；
* `Carrier`根据一定规则选择`GameServer`，然后设置`Constraint`，通知应用程序下线该副本；
* RoomAssign下线对应的`GameServer`，设置Condition `offline=true`，表示不再分配新的玩家到该`GameServer`；
* `GameServer`等到没有玩家之后，再设置Condition `no-player=true`；
* `Carrier controller`删除`offline=true && no-player=true`的`GameServer`。

注意，上面的`Condition`，比如`offline`，`no-player`可由业务自己定义。