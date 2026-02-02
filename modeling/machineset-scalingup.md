# MachineSet Reconciler - ScalingUp Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `ScalingUp`

## Overview

The `ScalingUp` condition on a MachineSet indicates whether the MachineSet is actively scaling up (creating new Machines) to reach the desired replica count. This condition helps operators understand the current scaling state and any blockers preventing scale-up.

---

## Condition Transition Table — `ScalingUp` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingUp`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachinesForMachineSetSucceeded == false` | No-op; internal error listing machines | `ScalingUp=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.replicas == nil` | No-op; replicas not yet set | `ScalingUp=Unknown; Reason=WaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && !DeletionTimestamp.IsZero()` | No-op; deleting, no need to scale up | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && DeletionTimestamp.IsZero() && missingReferences == ""` | No-op; desired replicas already met | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas >= desiredReplicas && DeletionTimestamp.IsZero() && missingReferences != ""` | No-op; at desired but warn about missing refs | `ScalingUp=False; Reason=NotScalingUp; Message="Scaling up would be blocked because <missingReferences>"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && no blockers` | Create new Machines to reach desiredReplicas | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && bootstrapObjectNotFound` | No-op; blocked, cannot create machines | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas is blocked because:\n* spec.template.spec.bootstrap.configRef references a <Kind> that does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && infrastructureObjectNotFound` | No-op; blocked, cannot create machines | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas is blocked because:\n* spec.template.spec.infrastructureRef references a <Kind> that does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `currentReplicas < desiredReplicas && preflightCheckErrors` | No-op; blocked by preflight checks | `ScalingUp=True; Reason=ScalingUp; Message="Scaling up from <current> to <desired> replicas is blocked because:\n* <preflight error messages>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Desired Replicas During Deletion**: When `DeletionTimestamp` is set, `desiredReplicas` is treated as 0 regardless of `spec.replicas`. This ensures the condition correctly reflects that no scale-up is needed when deleting.

2. **Missing References**: The condition surfaces warnings when referenced Bootstrap or Infrastructure templates are not found. These blocks prevent new Machine creation.

3. **Preflight Checks**: Before creating machines, the controller runs preflight checks. Failures in these checks are surfaced in the condition message as blockers.

4. **Current Replicas**: `currentReplicas` is the count of Machines currently owned by the MachineSet, calculated from the actual list of Machines.

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineSet and its owned Machines.

6. **Related Conditions**: 
   - `ScalingDown` indicates whether the MachineSet is scaling down
   - `MachinesReady` indicates whether owned Machines are ready
   - `MachinesUpToDate` indicates whether owned Machines are up-to-date
