---
bookFlatSection: true
title: "Frequently Asked Questions"
weight: 7
---

## Squad保留多少个历史版本？

`Squad.Spec.RevisionHistoryLimit`控制Squad历史版本的数量，默认为10个。

## Squad扩容时创建副本是串行，还是并发创建？

Squad通过GameServerSet控制GameServer副本的数量。在增加副本时，是由多个(16) Goroutine并发创建GameServer:

```golang
func (c *Controller) addMoreGameServers(gsSet *carrierv1alpha1.GameServerSet, count int) error {
	klog.Infof("Adding more gameservers: %v, count: %v", gsSet.Name, count)

	return parallelize(newGameServersChannel(count, gsSet), maxCreationParalellism, func(gs *carrierv1alpha1.GameServer) error {
		gameservers.ApplyDefaults(gs)
		gs, err := c.gameServerGetter.GameServers(gs.Namespace).Create(gs)
		if err != nil {
			return errors.Wrapf(err, "error creating gameserver for gameserverset %s", gsSet.Name)
		}

		c.stateCache.forGameServerSet(gsSet).created(gs)
		c.recorder.Eventf(gsSet, corev1.EventTypeNormal, "SuccessfulCreate", "Created gameserver: %s", gs.Name)
		return nil
	})
}
```

## Squad更新时检查DeletableGates

当Squad在缩容删除GameServer地，会检查`DeletableGates Condition`，只有当所有`DeletableGates Condition`都会True时（由应用程序控制），才会删除对应的GameServer：

```golang
// IsDeletable returns false if the server is currently not deletable
func IsDeletable(gs *carrierv1alpha1.GameServer) bool {
	if IsBeforeReady(gs) {
		return true
	}
	condMap := make(map[string]carrierv1alpha1.ConditionStatus, len(gs.Status.Conditions))
	for _, condition := range gs.Status.Conditions {
		condMap[string(condition.Type)] = condition.Status
	}
	for _, gate := range gs.Spec.DeletableGates {
		if v, ok := condMap[gate]; !ok || v != carrierv1alpha1.ConditionTrue {
			return false
		}
	}
	return true
}
```

同样，Squad支持优雅更新，可以在Squad的annotations指定`carrier.ocgi.dev/graceful-update: "true"`。这样，当Squad做更新删除GameServer时，也会检查`DeletableGates Condition`。
