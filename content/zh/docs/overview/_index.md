---
title: Overview
type: docs
weight: 1
---

## Introduction

`OCGI(Open Cloud-native Game-application Initiative)`是由腾讯游戏计算资源团队创建的一个开源项目，主要解决游戏后端服务在Kubernetes集群上运行和自动伸缩的问题。

对于竞技类游戏，比如，FPS，MMO和MOBA等在线多人游戏，都有Dedicated Game Server（简称DS），用于用户玩家进行游戏对局。

一般来说，DS是有状态的应用程序，可将完整的游戏模拟保留在内存中。 但是与其他有状态应用程序（例如数据库）不同，它们的生命周期相对比较短。 DS进程一般持续数分钟或数小时，随着游戏对局结束，就会退出。

Kubernetes的[Statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)无法很好地管理此类应用程序。 因此，我们创建了OCGI项目，让游戏服务器可以更好的在Kubernetes上运行和自动伸缩。


## OCGI组件

OCGI主要包括下面一些组件：

- GameServer

  `GameServer`代表单个游戏后端Server。它基于[`K8s Pod`](https://kubernetes.io/docs/concepts/workloads/pods/)实现，是对`K8s Pod`的进一步抽象。

- Squad

  `Squad`代表一组游戏后端Server(`GameServer`)，它们具有相同的资源配置，并由`Carrier controller`维持该组GameServer在指定的副本数量。它控制该组GameServer的发布和更新。

  `GameServer`、`Squad`由[Carrier controller](https://github.com/ocgi/carrier)统一管理。

- GeneralPodAutoscaler

  [GeneralPodAutoscaler](https://github.com/ocgi/generalpodautoscaler)是自动弹性伸缩控制器，它根据用户指定的策略，动态调整`Squad`的`GameServer`副本数量。

`Squad`和`GeneralPodAutoscaler`提供了一些扩展和交互机制，变更，或者扩缩容时，`GameServer`可以更加优雅的退出，避免对游戏玩家的影响。

## 应用架构案例

![Application archtecture](/images/application_archtecture.png)

- MatchMaker: 负责对局匹配（由业务侧开发）
- Dscenter: 负责Dedicated Server管理和分配（由业务侧开发）
- Dedicated Server: 对应一个`GameServer`，管理多个ds进程，并上报ds信息到Dscenter。`Dedicated Server`和`Carrier-SDK`作为一个整体，部署在同一个`K8s Pod`
- Carrier Controller: 管理一组GameServer (创建、更新、删除)，维持DS集群在一定的副本数量
- AutoScaler: 根据业务metric、event、time等对DS集群的副本数进行计算和调整

