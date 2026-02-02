# Cluster Reconciler - WorkerMachinesReady Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `WorkerMachinesReady`

## Overview

The `WorkerMachinesReady` condition on a Cluster reflects the readiness status of worker machines. It aggregates the `Ready` condition from all worker Machines belonging to the cluster (excluding MachinePool machines).

---

## Condition Transition Table — `WorkerMachinesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `WorkerMachinesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `!getDescendantsSucceeded` (error listing descendants) | No-op; surface internal error | `WorkerMachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(workerMachines) == 0` | No-op; no worker machines exist | `WorkerMachinesReady=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all worker Machines have Ready=True` | Aggregate conditions from all worker machines | `WorkerMachinesReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any worker Machine has Ready=False` | Aggregate conditions; surface issues | `WorkerMachinesReady=False; Reason=NotReady; Message="<aggregated issues from worker Machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any worker Machine has Ready=Unknown` | Aggregate conditions; surface unknown status | `WorkerMachinesReady=Unknown; Reason=ReadyUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `WorkerMachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `Ready` conditions from all worker `Machine` objects belonging to the cluster.

2. **Worker Machine Selection**: Worker machines are identified by NOT having the `cluster.x-k8s.io/control-plane` label. MachinePool machines (identified by `cluster.x-k8s.io/pool-name` label) are excluded.

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotReady` for False status (issue reason)
   - `ReadyUnknown` for Unknown status
   - `Ready` for True status (info reason)

4. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

5. **No Replicas Case**: When no worker machines exist, the condition is set to `True` with reason `NoReplicas`.
