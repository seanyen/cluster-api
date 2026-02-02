# KubeadmControlPlane Reconciler - ScalingDown Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `ScalingDown`

## Overview

The `ScalingDown` condition on a KubeadmControlPlane indicates whether the control plane is scaling down (removing replicas). This condition has **negative polarity** (True = scaling down, False = stable).

---

## Condition Transition Table — `ScalingDown` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingDown`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.replicas == nil` | No-op; waiting for replicas to be set | `ScalingDown=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) <= *spec.replicas && deletionTimestamp.IsZero()` | No-op; not scaling down | `ScalingDown=False; Reason=NotScalingDown; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > *spec.replicas || (!deletionTimestamp.IsZero() && len(machines) > 0)` (scaling down with preflight checks pass) | Scaling down in progress | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > desiredReplicas && (preflightChecks fail || staleMachines exist)` | Scaling down blocked | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas is blocked because:\n<reasons>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Scaling down (transitional state)
   - `False` = Not scaling down (stable state)

2. **Preflight Checks**: Scaling down can be blocked by preflight checks:
   - Etcd cluster is not healthy
   - Control plane components are not healthy
   - Cluster is paused

3. **Stale Machines**: Machines marked as "stale" (stuck in deletion) are surfaced in the blocking message.

4. **Deletion Handling**: When `deletionTimestamp` is set, desired replicas is treated as 0 (scaling down to zero).

5. **Watch-Driven**: The reconciler is watch-driven; no explicit `requeueAfter` is used for this condition.
