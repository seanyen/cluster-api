# KubeadmControlPlane Reconciler - MachinesUpToDate Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `MachinesUpToDate`

## Overview

The `MachinesUpToDate` condition on a KubeadmControlPlane indicates whether all owned Machines are running the desired configuration. It aggregates the `UpToDate` condition from all Machines owned by the KubeadmControlPlane.

---

## Condition Transition Table — `MachinesUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machines) == 0` (after filtering) | No-op; no machines exist | `MachinesUpToDate=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all machines have UpToDate=True` | Aggregate conditions from all machines | `MachinesUpToDate=True; Reason=MachinesUpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=False` | Aggregate conditions; surface issues | `MachinesUpToDate=False; Reason=MachinesNotUpToDate; Message="<aggregated issues from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=Unknown` | Aggregate conditions; surface unknown status | `MachinesUpToDate=Unknown; Reason=MachinesUpToDateUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `MachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `UpToDate` conditions from all `Machine` objects owned by the KubeadmControlPlane.

2. **Machine Filtering**: Only considers Machines that either:
   - Have an `UpToDate` condition already set, OR
   - Are older than 10 seconds (to avoid flickering after new Machine creation)

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `MachinesNotUpToDate` for False status (issue reason)
   - `MachinesUpToDateUnknown` for Unknown status
   - `MachinesUpToDate` for True status (info reason)

4. **No Replicas Case**: When no machines exist (after filtering), the condition is set to `True` with reason `NoReplicas`.

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
