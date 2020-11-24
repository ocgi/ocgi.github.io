---
title: Overview
type: docs
weight: 1
---

## Introduction

OCGI (Open Cloud-native Game-application Initiative) is a opensource project created by [Tencent Games](https://game.qq.com) Computing Resources Team, which mainly solves the problems when running and scaling game server on Kubernetes cluster.

Generally speaking, the online multiplayer games such as competitive [FPS](https://en.wikipedia.org/wiki/First-person_shooter)s and [MOBA](https://en.wikipedia.org/wiki/Multiplayer_online_battle_arena)s, require a [dedicated game server(DS)](https://en.wikipedia.org/wiki/Game_server#Dedicated_server) which simulating game worlds, and players connect to the server with separate client programs, then playing within it. 

Dedicated game servers are stateful applications that retain the full game simulation in memory. But unlike other stateful applications, such as databases, they have a short lifetime. Rather than running for months or years, a dedicated game server process will exit when a game is over, which usually lasts a few minutes or hours.

The Kubernetes [Statefulset](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) workload does not manage such applications well. So we developed this project to make game servers runing and scaling better on Kubernetes. 

## OCGI Overview

OCGI consists of several Kubernetes controllers for game server application management.

- GameServer

  GameServer represents a single game server. It is based on K8s [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) implementation and is a further abstraction of the Pod.

- Squad

  Squad represents a group of game servers (GameServer), they have the same resource configuration, and the controller maintains the specified number of replicas of the group of GameServers. It controls the updating and scaling of the group of GameServers.

  `GameServer` and `Squad` are managed by the [Carrier](https://github.com/ocgi/carrier) controller. Carrier communicates with the game server through the SDK, and game server can notify the Carrier when no player whthin it, then the Carrier can delete the Pod safely. Conversely, when scaling down the Kubernetes cluster, Carrier can also notify the game server through the SDK, which allows the Carrier to better running and scaling the game server.

- GeneralPodAutoscaler

  `GeneralPodAutoscaler` is an auto-scaling controller, which dynamically adjusts the number of GameServer replicas of Squad according to the strategy specified by the application.

`Squad` and `GeneralPodAutoscaler` provide some extension mechanisms. When updating or scaling, GameServer can exit more gracefully, avoiding the impact on game players.

## Main Features

- Application Interactive Update

  Support application interactive update, delete the replica only after the application confirmed. This is very important for game server, but the Deployment and Statefulset can not support this.

- In-place Update

  Support in-place update image, and the Pod will not be recreated. The local cache data GameServer can be retained when updating.

- Multiple Pod Auto-scaling Strategies

  Support multiple modes (resource metrics/custom metrics/timing/events/webhook) Pod auto-scaling strategies. In many cases, the application wants to specify the number of service replicas by itself, which can be achieved through webhooks.

- Application-defined Scaling-down Order

  Application can define the scaling-down order of replicas. For example, application can choose the game server replica with least user-accessed to delete. This can not only reduce the cost of scaling-down, but also improve the resource utilization.

- Better Cluster Auto-scaling

  It can be seamlessly integrated with the cluster auto-scaling. Based on the application-confirmed mechanism, can choose any replica to delete after application confirmed when cluster scaling-down.

## Application architecture based on OCGI

![Application archtecture](/images/application_archtecture.png)

- MatchMaker

  Responsible for matchmaking (developed by the application)

- Dscenter
 
  esponsible for Dedicated Server management and allocation (developed by the application)

- Dedicated Server

  Corresponds to a GameServer, manages multiple ds processes, and reports ds information to `Dscenter`. `Dedicated Server` and `Carrier-SDK` as a whole are deployed in the same K8s Pod

- Carrier Controller

  Manage a group of GameServers (create, update, delete) and maintain a certain number of replicas of the DS cluster

- Autoscaler

  Calculate and adjust the number of replicas of the DS cluster according to application metrics, events, time, etc.
