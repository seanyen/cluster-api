# MachineDeployment Reconciler - MachinesUpToDate Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `MachinesUpToDate`

## Overview

The `MachinesUpToDate` condition on a MachineDeployment indicates whether all Machines in the deployment are running the desired configuration. It aggregates the `UpToDate` condition from all Machines owned by the MachineDeployment (across all MachineSets).

---

## Condition Transition Table — `MachinesUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machines) == 0` (after filtering) | No-op; no machines exist | `MachinesUpToDate=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all machines have UpToDate=True` | Aggregate conditions from all machines | `MachinesUpToDate=True; Reason=UpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=False` | Aggregate conditions; surface issues | `MachinesUpToDate=False; Reason=NotUpToDate; Message="<aggregated issues from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=Unknown` | Aggregate conditions; surface unknown status | `MachinesUpToDate=Unknown; Reason=UpToDateUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `MachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `UpToDate` conditions from all `Machine` objects owned by the MachineDeployment (through its MachineSets).

2. **Machine Filtering**: Only considers Machines that either:
   - Have an `UpToDate` condition already set, OR
   - Are older than 10 seconds (to avoid flickering after new Machine creation)

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotUpToDate` for False status (issue reason)
   - `UpToDateUnknown` for Unknown status
   - `UpToDate` for True status (info reason)

4. **No Replicas Case**: When no machines exist (after filtering), the condition is set to `True` with reason `NoReplicas`.

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.

6. **Relationship to RollingOut**: This condition is related to but distinct from `RollingOut`:
   - `MachinesUpToDate` reports aggregate status of all machines' UpToDate conditions
   - `RollingOut` specifically tracks active rollout progress with count and reasons
