# MachineDeployment Reconciler - ScalingDown Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `ScalingDown`

## Overview

The `ScalingDown` condition on a MachineDeployment indicates whether the MachineDeployment is actively scaling down (deleting Machines via MachineSets) to reach the desired replica count. This condition helps operators understand the current scaling state and identify any issues with machines being deleted.

---

## Condition Transition Table — `ScalingDown` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingDown`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachineSetsForDeploymentSucceeded == false` | No-op; internal error listing MachineSets | `ScalingDown=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.replicas == nil` | No-op; replicas not yet set | `ScalingDown=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas > desiredReplicas && no stale machines` | Scale down via MachineSet | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas > desiredReplicas && staleMachines exist` | Scale down via MachineSet; report stale machines | `ScalingDown=True; Reason=ScalingDown; Message="Scaling down from <current> to <desired> replicas\n* <stale machine info>"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas <= desiredReplicas` | No-op; desired replicas reached | `ScalingDown=False; Reason=NotScalingDown; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Desired Replicas During Deletion**: When `DeletionTimestamp` is set on the MachineDeployment, `desiredReplicas` is treated as 0 regardless of `spec.replicas`. This ensures all machines get deleted.

2. **Current Replicas Calculation**: `currentReplicas` is calculated using `mdutil.GetActualReplicaCountForMachineSets()` which sums replicas across all MachineSets owned by the MachineDeployment.

3. **Stale Machines**: The condition message includes information about stale machines (machines that have been deleting for an extended period) to help operators identify problematic deletions.

4. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineDeployment, its owned MachineSets, and the Machines.

5. **Related Conditions**: 
   - `ScalingUp` indicates whether the MachineDeployment is scaling up
   - `Deleting` indicates whether the MachineDeployment itself is being deleted
   - `MachinesReady` indicates whether owned Machines are ready
