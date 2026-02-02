# MachineSet Reconciler - MachinesUpToDate Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `MachinesUpToDate`

## Overview

The `MachinesUpToDate` condition on a MachineSet indicates whether all owned Machines are running the desired configuration. It aggregates the `UpToDate` condition from all Machines owned by the MachineSet.

---

## Condition Transition Table — `MachinesUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `!getAndAdoptMachinesForMachineSetSucceeded` (error listing machines) | No-op; surface internal error | `MachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) == 0` (after filtering) | No-op; no machines exist | `MachinesUpToDate=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all machines have UpToDate=True` | Aggregate conditions from all machines | `MachinesUpToDate=True; Reason=UpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=False` | Aggregate conditions; surface issues | `MachinesUpToDate=False; Reason=NotUpToDate; Message="<aggregated issues from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `any machine has UpToDate=Unknown` | Aggregate conditions; surface unknown status | `MachinesUpToDate=Unknown; Reason=UpToDateUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `MachinesUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregation Source**: The condition aggregates `UpToDate` conditions from all `Machine` objects owned by the MachineSet.

2. **Machine Filtering**: Only considers Machines that either:
   - Have an `UpToDate` condition already set, OR
   - Are older than 10 seconds (to avoid flickering after new Machine creation)

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with custom reason computation:
   - `NotUpToDate` for False status (issue reason)
   - `UpToDateUnknown` for Unknown status
   - `UpToDate` for True status (info reason)

4. **No Replicas Case**: When no machines exist (after filtering), the condition is set to `True` with reason `NoReplicas`.

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
