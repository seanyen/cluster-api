# MachineSet Reconciler - ScalingDown Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `ScalingDown`

## Overview

The `ScalingDown` condition on a MachineSet indicates whether the MachineSet is actively scaling down (deleting Machines) to reach the desired replica count. This condition helps operators understand the current scaling state and identify any issues with machines being deleted.

---

## Condition Transition Table — `ScalingDown` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingDown`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachinesForMachineSetSucceeded == false` | No-op; internal error listing machines | `ScalingDown=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.replicas == nil` | No-op; replicas not yet set | `ScalingDown=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas > desiredReplicas && no stale machines` | Delete excess Machines | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas > desiredReplicas && staleMachines exist` | Delete excess Machines; report stale machines | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas\n* <stale machine info>"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas <= desiredReplicas` | No-op; desired replicas reached | `ScalingDown=False; Reason=NotScalingDown; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Desired Replicas During Deletion**: When `DeletionTimestamp` is set on the MachineSet, `desiredReplicas` is treated as 0 regardless of `spec.replicas`. This ensures all machines get deleted.

2. **Pending Acknowledgment**: Machines with the `PendingAcknowledgeMoveAnnotation` are not counted towards `currentReplicas`. This prevents counting machines that are being moved during cluster migration.

3. **Stale Machines**: The condition message includes information about stale machines (machines that have been deleting for an extended period) to help operators identify problematic deletions.

4. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineSet and its owned Machines.

5. **Related Conditions**: 
   - `ScalingUp` indicates whether the MachineSet is scaling up
   - `MachinesReady` indicates whether owned Machines are ready
   - `Deleting` indicates whether the MachineSet itself is being deleted
