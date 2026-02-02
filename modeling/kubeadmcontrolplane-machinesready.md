# KubeadmControlPlane Reconciler - MachinesReady Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `MachinesReady`

## Overview

The `MachinesReady` condition on a KubeadmControlPlane indicates whether all owned Machines are ready. It aggregates the `Ready` condition from all Machines owned by the KubeadmControlPlane.

---

## Condition Transition Table — `MachinesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machines) == 0` | No-op; no machines exist | `MachinesReady=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all machines have Ready=True` | Aggregate conditions from all machines | `MachinesReady=True; Reason=MachinesReady; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any machine has Ready=False` | Aggregate conditions; surface issues | `MachinesReady=False; Reason=MachinesNotReady; Message="<aggregated issues from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any machine has Ready=Unknown` | Aggregate conditions; surface unknown status | `MachinesReady=Unknown; Reason=MachinesReadyUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `MachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `Ready` conditions from all `Machine` objects owned by the KubeadmControlPlane.

2. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `MachinesNotReady` for False status (issue reason)
   - `MachinesReadyUnknown` for Unknown status
   - `MachinesReady` for True status (info reason)

3. **No Replicas Case**: When no machines exist, the condition is set to `True` with reason `NoReplicas`.

4. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
