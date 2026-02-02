# MachineDeployment Reconciler - Deleting Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `Deleting`

## Overview

The `Deleting` condition on a MachineDeployment indicates whether the MachineDeployment is being deleted. This condition tracks the deletion progress and surfaces information about remaining MachineSets and Machines that need to be cleaned up.

---

## Condition Transition Table — `Deleting` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Deleting`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachineSetsForDeploymentSucceeded == false` | No-op; internal error listing MachineSets/Machines | `Deleting=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `DeletionTimestamp.IsZero()` | No-op; not deleting | `Deleting=False; Reason=NotDeleting; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) == 1 && no stale machines` | Continue deletion; delete remaining machine | `Deleting=True; Reason=Deleting; Message="Deleting 1 Machine"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) > 1 && no stale machines` | Continue deletion; delete remaining machines | `Deleting=True; Reason=Deleting; Message="Deleting <n> Machines"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) > 0 && staleMachines exist` | Continue deletion; report stale machines | `Deleting=True; Reason=Deleting; Message="Deleting <n> Machine(s)\n* <stale machine info>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) == 0 && len(machineSets) > 0` | Continue deletion; waiting for MachineSets cleanup | `Deleting=True; Reason=Deleting; Message="Deleting <n> MachineSets"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && len(machines) == 0 && len(machineSets) == 0` | Remove finalizer; complete deletion | `Deleting=True; Reason=Deleting; Message="Deletion completed"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Finalizer**: The MachineDeployment controller uses a finalizer to ensure all owned MachineSets and their Machines are deleted before the MachineDeployment itself is removed.

2. **Deletion Hierarchy**: The deletion order is:
   - MachineDeployment triggers deletion of MachineSets
   - MachineSets trigger deletion of Machines
   - Once all Machines are deleted, MachineSets can be garbage collected
   - Once all MachineSets are gone, MachineDeployment finalizer is removed

3. **Stale Machines**: The condition message includes information about stale machines (machines that have been deleting for more than 15 minutes) to help operators identify problematic deletions (e.g., stuck drains).

4. **MachineSets Without Machines**: There's a brief transition period where MachineSets may exist without Machines while finalizers are being removed.

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineDeployment, MachineSets, and Machines.

6. **Related Conditions**: 
   - `ScalingDown` indicates active scale-down
   - `MachinesReady` indicates machine health during deletion
