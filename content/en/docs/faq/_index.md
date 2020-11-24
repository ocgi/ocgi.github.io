---
bookFlatSection: true
title: "Frequently Asked Questions"
weight: 7
---

## How many historical versions does Squad keep?

`Squad.Spec.RevisionHistoryLimit` controls the number of historical versions of Squad, and the default is 10.

## When Squad scaling-up, is the replicas created serially or concurrently?

Squad controls the number of copies of GameServer through GameServerSet. When adding copies, multiple (16) Goroutines concurrently create GameServer:

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

## Check DeleteableGates when Squad is updated

When Squad is scaling-down and deleting GameServer, it will check the `DeleteableGates` condition. Only when all the `DeleteableGates` condition will be `True` (controlled by the application), will the corresponding GameServer be deleted:

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

Similarly, Squad supports graceful updates. You can specify `carrier.ocgi.dev/graceful-update: "true"` in Squad's annotations. In this way, when Squad updates and deletes GameServer, it will also check the DeleteableGates condition.