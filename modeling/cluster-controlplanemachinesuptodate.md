# Cluster Reconciler - ControlPlaneMachinesUpToDate Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ControlPlaneMachinesUpToDate`

## Overview

The `ControlPlaneMachinesUpToDate` condition on a Cluster reflects whether all control plane machines are running the desired configuration. It aggregates the `UpToDate` condition from all control plane Machines belonging to the cluster.

---

## Condition Transition Table — `ControlPlaneMachinesUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneMachinesUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `!getDescendantsSucceeded` (error listing descendants) | No-op; surface internal error | `ControlPlaneMachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(controlPlaneMachines) == 0` (after filtering) | No-op; no control plane machines exist | `ControlPlaneMachinesUpToDate=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all control plane Machines have UpToDate=True` | Aggregate conditions from all control plane machines | `ControlPlaneMachinesUpToDate=True; Reason=UpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any control plane Machine has UpToDate=False` | Aggregate conditions; surface issues | `ControlPlaneMachinesUpToDate=False; Reason=NotUpToDate; Message="<aggregated issues from control plane Machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any control plane Machine has UpToDate=Unknown` | Aggregate conditions; surface unknown status | `ControlPlaneMachinesUpToDate=Unknown; Reason=UpToDateUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `ControlPlaneMachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `UpToDate` conditions from all control plane `Machine` objects belonging to the cluster.

2. **Machine Filtering**: Only considers Machines that either:
   - Have an `UpToDate` condition already set, OR
   - Are older than 10 seconds (to avoid flickering after new Machine creation)

3. **Control Plane Machine Selection**: Control plane machines are identified by having the `cluster.x-k8s.io/control-plane` label or being owned by the control plane provider.

4. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotUpToDate` for False status (issue reason)
   - `UpToDateUnknown` for Unknown status
   - `UpToDate` for True status (info reason)

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

6. **No Replicas Case**: When no control plane machines exist (after filtering), the condition is set to `True` with reason `NoReplicas`.
