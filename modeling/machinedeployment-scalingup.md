# MachineDeployment Reconciler - ScalingUp Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `ScalingUp`

## Overview

The `ScalingUp` condition on a MachineDeployment indicates whether the MachineDeployment is actively scaling up (creating new Machines via MachineSets) to reach the desired replica count. This condition helps operators understand the current scaling state and any blockers preventing scale-up.

---

## Condition Transition Table — `ScalingUp` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingUp`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachineSetsForDeploymentSucceeded == false` | No-op; internal error listing MachineSets | `ScalingUp=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.replicas == nil` | No-op; replicas not yet set | `ScalingUp=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && !DeletionTimestamp.IsZero()` | No-op; deleting, no need to scale up | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && DeletionTimestamp.IsZero() && missingReferences == ""` | No-op; desired replicas already met | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && DeletionTimestamp.IsZero() && missingReferences != ""` | No-op; at desired but warn about missing refs | `ScalingUp=False; Reason=NotScalingUp; Message="Scaling up would be blocked <missingReferences>"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && no blockers` | Scale up via MachineSet | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && missingReferences != ""` | No-op; blocked, cannot create machines | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas is blocked <missingReferences>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Desired Replicas During Deletion**: When `DeletionTimestamp` is set, `desiredReplicas` is treated as 0 regardless of `spec.replicas`. This ensures the condition correctly reflects that no scale-up is needed when deleting.

2. **Missing References**: The condition surfaces warnings when referenced Bootstrap or Infrastructure templates are not found. These blocks prevent new Machine creation.

3. **Current Replicas Calculation**: `currentReplicas` is calculated using `mdutil.GetActualReplicaCountForMachineSets()` which sums replicas across all MachineSets owned by the MachineDeployment.

4. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineDeployment, its owned MachineSets, and the Machines.

5. **Related Conditions**: 
   - `ScalingDown` indicates whether the MachineDeployment is scaling down
   - `RollingOut` indicates whether a rollout is in progress
   - `MachinesReady` indicates whether owned Machines are ready
   - `Available` indicates overall availability
