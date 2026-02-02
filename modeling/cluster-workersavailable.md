# Cluster Reconciler - WorkersAvailable Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `WorkersAvailable`

## Overview

The `WorkersAvailable` condition on a Cluster reflects the availability status of worker nodes. It aggregates the `Available` condition from all MachineDeployments and MachinePools belonging to the cluster.

---

## Condition Transition Table — `WorkersAvailable` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `WorkersAvailable`) | Next reconcile (Return) |
|---|---|---|---|
| `!getDescendantsSucceeded` (error listing descendants) | No-op; surface internal error | `WorkersAvailable=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinePools) == 0 && len(machineDeployments) == 0` | No-op; no workers defined | `WorkersAvailable=True; Reason=NoWorkers; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all MachineDeployments and MachinePools have Available=True` | Aggregate conditions from all workers | `WorkersAvailable=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any MachineDeployment or MachinePool has Available=False` | Aggregate conditions; surface issues | `WorkersAvailable=False; Reason=NotAvailable; Message="<aggregated issues from MachineDeployments/MachinePools>"; ObservedGeneration=metadata.generation` | `none` |
| `any MachineDeployment or MachinePool has Available=Unknown` | Aggregate conditions; surface unknown status | `WorkersAvailable=Unknown; Reason=AvailableUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `WorkersAvailable=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `Available` conditions from:
   - All `MachineDeployment` objects owned by the cluster
   - All `MachinePool` objects owned by the cluster

2. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotAvailable` for False status
   - `AvailableUnknown` for Unknown status
   - `Available` for True status

3. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

4. **No Workers Case**: When no MachineDeployments or MachinePools exist, the condition is set to `True` with reason `NoWorkers`.
