---
title: "GameServer autoscaling workflow"
weight: 4
---

The main flow of `GameServer` autoscaling is as follows:

![autoscaler archtecture](/images/autoscaler_archtecture.png)

* The Autoscaler controller calculates a reasonable number of copies of Squad according to the metric information of GameServer;

* Carrier selects GameServer according to certain rules, and then sets Constraint to notify the application to offline the copy;

* For the GameServer corresponding to RoomAssign offline, set condition `offline=true`, which means that no new players will be assigned to the GameServer;

* GameServer waits until there are no players before setting condition `no-player=true`;

* The Carrier controller deletes the GameServer with `offline=true && no-player=true`.

Note that the above condition, such as `offline`, `no-player`, can be defined by the application itself.
