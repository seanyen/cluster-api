# Cluster Reconciler - ControlPlaneMachinesReady Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ControlPlaneMachinesReady`

## Overview

The `ControlPlaneMachinesReady` condition on a Cluster reflects the readiness status of control plane machines. It aggregates the `Ready` condition from all control plane Machines belonging to the cluster.

---

## Condition Transition Table — `ControlPlaneMachinesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneMachinesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `!getDescendantsSucceeded` (error listing descendants) | No-op; surface internal error | `ControlPlaneMachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(controlPlaneMachines) == 0` | No-op; no control plane machines exist | `ControlPlaneMachinesReady=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all control plane Machines have Ready=True` | Aggregate conditions from all control plane machines | `ControlPlaneMachinesReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any control plane Machine has Ready=False` | Aggregate conditions; surface issues | `ControlPlaneMachinesReady=False; Reason=NotReady; Message="<aggregated issues from control plane Machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any control plane Machine has Ready=Unknown` | Aggregate conditions; surface unknown status | `ControlPlaneMachinesReady=Unknown; Reason=ReadyUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `ControlPlaneMachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `Ready` conditions from all control plane `Machine` objects belonging to the cluster.

2. **Control Plane Machine Selection**: Control plane machines are identified by having the `cluster.x-k8s.io/control-plane` label or being owned by the control plane provider.

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotReady` for False status (issue reason)
   - `ReadyUnknown` for Unknown status
   - `Ready` for True status (info reason)

4. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

5. **No Replicas Case**: When no control plane machines exist, the condition is set to `True` with reason `NoReplicas`.
