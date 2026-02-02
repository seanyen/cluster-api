# MachineSet Reconciler - Deleting Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `Deleting`

## Overview

The `Deleting` condition on a MachineSet indicates whether the MachineSet is being deleted. This condition tracks the deletion progress and surfaces information about remaining Machines that need to be cleaned up.

---

## Condition Transition Table — `Deleting` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Deleting`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachinesForMachineSetSucceeded == false` | No-op; internal error listing machines | `Deleting=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `DeletionTimestamp.IsZero()` | No-op; not deleting | `Deleting=False; Reason=NotDeleting; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) == 1 && no stale machines` | Continue deletion; delete remaining machine | `Deleting=True; Reason=Deleting; Message="Deleting 1 Machine"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) > 1 && no stale machines` | Continue deletion; delete remaining machines | `Deleting=True; Reason=Deleting; Message="Deleting <n> Machines"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) > 0 && staleMachines exist` | Continue deletion; report stale machines | `Deleting=True; Reason=Deleting; Message="Deleting <n> Machine(s)\n* <stale machine info>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) == 0` | Remove finalizer; complete deletion | `Deleting=True; Reason=Deleting; Message="Deletion completed"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Finalizer**: The MachineSet controller uses a finalizer to ensure all owned Machines are deleted before the MachineSet itself is removed.

2. **Stale Machines**: The condition message includes information about stale machines (machines that have been deleting for an extended period) to help operators identify problematic deletions.

3. **Deletion Order**: The MachineSet must delete all its Machines before it can be fully deleted. The condition tracks this progress.

4. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineSet and its owned Machines.

5. **Related Conditions**: 
   - `ScalingDown` indicates the scaling down state
   - `MachinesReady` indicates whether owned Machines are ready
