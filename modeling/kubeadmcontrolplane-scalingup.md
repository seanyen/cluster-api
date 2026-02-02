# KubeadmControlPlane Reconciler - ScalingUp Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `ScalingUp`

## Overview

The `ScalingUp` condition on a KubeadmControlPlane indicates whether the control plane is scaling up (adding replicas). This condition has **negative polarity** (True = scaling up, False = stable).

---

## Condition Transition Table — `ScalingUp` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingUp`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.replicas == nil` | No-op; waiting for replicas to be set | `ScalingUp=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) >= *spec.replicas && !deletionTimestamp.IsZero()` (deleting) | No-op; not scaling up during deletion | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) >= *spec.replicas && infraMachineTemplateNotFound` | No-op; would be blocked | `ScalingUp=False; Reason=NotScalingUp; Message="Scaling up would be blocked because <kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) >= *spec.replicas` | No-op; not scaling up | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) < *spec.replicas && preflightChecks pass && infraMachineTemplate exists` | Scaling up in progress | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) < *spec.replicas && (preflightChecks fail || infraMachineTemplateNotFound)` | Scaling up blocked | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas is blocked because:\n<reasons>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Scaling up (transitional state)
   - `False` = Not scaling up (stable state)

2. **Preflight Checks**: Scaling up can be blocked by preflight checks:
   - Etcd cluster is not healthy
   - Control plane components are not healthy
   - Cluster is paused

3. **Missing Infrastructure Template**: If the infrastructure machine template reference doesn't exist, scaling is blocked and surfaced in the message.

4. **Deletion Handling**: When `deletionTimestamp` is set, desired replicas is treated as 0.

5. **Watch-Driven**: The reconciler is watch-driven; no explicit `requeueAfter` is used for this condition.
