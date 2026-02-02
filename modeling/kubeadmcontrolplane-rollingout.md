# KubeadmControlPlane Reconciler - RollingOut Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `RollingOut`

## Overview

The `RollingOut` condition on a KubeadmControlPlane indicates whether a rolling update is in progress. It is derived by aggregating the `UpToDate` condition from all owned Machines. This condition has **negative polarity** (True = rolling out, False = stable).

---

## Condition Transition Table — `RollingOut` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RollingOut`) | Next reconcile (Return) |
|---|---|---|---|
| `no machines have UpToDate=False` | No-op; no rollout in progress | `RollingOut=False; Reason=NotRollingOut; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `one or more machines have UpToDate=False` | Aggregate rollout reasons from machines | `RollingOut=True; Reason=RollingOut; Message="Rolling out <n> not up-to-date replicas\n<reasons>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Rolling out (transitional state)
   - `False` = Not rolling out (stable state)

2. **Rolling Out Detection**: A machine is considered "rolling out" when its `UpToDate` condition is `False`.

3. **Rollout Reasons**: The condition message includes all unique reasons why machines are not up-to-date:
   - Version changes (sorted first)
   - Spec changes
   - Infrastructure changes
   - Bootstrap config changes

4. **Message Aggregation**: Reasons are extracted from each machine's `UpToDate` condition message and deduplicated.

5. **Watch-Driven**: The reconciler is watch-driven; no explicit `requeueAfter` is used for this condition.
