# Cluster Reconciler - WorkerMachinesUpToDate Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `WorkerMachinesUpToDate`

## Overview

The `WorkerMachinesUpToDate` condition on a Cluster reflects whether all worker machines are running the desired configuration. It aggregates the `UpToDate` condition from all worker Machines belonging to the cluster (excluding MachinePool machines).

---

## Condition Transition Table — `WorkerMachinesUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `WorkerMachinesUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `!getDescendantsSucceeded` (error listing descendants) | No-op; surface internal error | `WorkerMachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(workerMachines) == 0` (after filtering) | No-op; no worker machines exist | `WorkerMachinesUpToDate=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all worker Machines have UpToDate=True` | Aggregate conditions from all worker machines | `WorkerMachinesUpToDate=True; Reason=UpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any worker Machine has UpToDate=False` | Aggregate conditions; surface issues | `WorkerMachinesUpToDate=False; Reason=NotUpToDate; Message="<aggregated issues from worker Machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any worker Machine has UpToDate=Unknown` | Aggregate conditions; surface unknown status | `WorkerMachinesUpToDate=Unknown; Reason=UpToDateUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `WorkerMachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `UpToDate` conditions from all worker `Machine` objects belonging to the cluster.

2. **Machine Filtering**: Only considers Machines that either:
   - Have an `UpToDate` condition already set, OR
   - Are older than 10 seconds (to avoid flickering after new Machine creation)

3. **Worker Machine Selection**: Worker machines are identified by NOT having the `cluster.x-k8s.io/control-plane` label. MachinePool machines (identified by `cluster.x-k8s.io/pool-name` label) are excluded.

4. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotUpToDate` for False status (issue reason)
   - `UpToDateUnknown` for Unknown status
   - `UpToDate` for True status (info reason)

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

6. **No Replicas Case**: When no worker machines exist (after filtering), the condition is set to `True` with reason `NoReplicas`.
