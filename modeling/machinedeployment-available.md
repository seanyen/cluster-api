# MachineDeployment Reconciler - Available Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `Available`

## Overview

The `Available` condition on a MachineDeployment indicates whether the deployment has sufficient available replicas to meet the minimum availability requirements. This takes into account the `maxUnavailable` setting from the rolling update strategy.

---

## Condition Transition Table — `Available` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Available`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachineSetsForDeploymentSucceeded == false` | No-op; cannot determine availability | `Available=Unknown; Reason=AvailableInternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.replicas == nil` | No-op; replicas not yet set | `Available=Unknown; Reason=AvailableWaitingForReplicasSet; Message="Waiting for spec.replicas set"; ObservedGeneration=metadata.generation` | `none` |
| `status.availableReplicas == nil` | No-op; available replicas not yet computed | `Available=Unknown; Reason=AvailableWaitingForAvailableReplicasSet; Message="Waiting for status.availableReplicas set"; ObservedGeneration=metadata.generation` | `none` |
| `status.availableReplicas >= minReplicasNeeded (spec.replicas - maxUnavailable)` | No-op; sufficient replicas available | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `status.availableReplicas < minReplicasNeeded && deletionTimestamp.IsZero()` | No-op; insufficient replicas | `Available=False; Reason=NotAvailable; Message="<N> available replicas, at least <M> required (spec.strategy.rollout.maxUnavailable is <X>, spec.replicas is <Y>)"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero() && status.availableReplicas < minReplicasNeeded` | No-op; deleting with insufficient replicas | `Available=False; Reason=NotAvailable; Message="Deletion in progress"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **minReplicasNeeded Calculation**: 
   - For RollingUpdate strategy: `minReplicasNeeded = spec.replicas - maxUnavailable`
   - For other strategies: `minReplicasNeeded = spec.replicas`

2. **maxUnavailable**: Derived from `spec.rollout.strategy.rollingUpdate.maxUnavailable`, which can be an absolute number or a percentage.

3. **Available Replicas Source**: The `status.availableReplicas` is computed by aggregating `status.availableReplicas` from all owned MachineSets via `mdutil.GetAvailableReplicaCountForMachineSets()`.

4. **Relationship to Machine Availability**: A Machine is counted as available when:
   - The Machine has `Available=True` condition
   - The Machine is part of an owned MachineSet

5. **Watch-Driven**: The reconciler is watch-driven via watches on owned MachineSets.
