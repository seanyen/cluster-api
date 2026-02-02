# Machine Reconciler - InfrastructureReady Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `InfrastructureReady`

## Overview

The `InfrastructureReady` condition on a Machine indicates whether the infrastructure (InfraMachine) has been successfully provisioned. This condition is primarily mirrored from the infrastructure provider's machine object.

---

## Condition Transition Table — `InfrastructureReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `InfrastructureReady`) | Next reconcile (Return) |
|---|---|---|---|
| `infraMachine exists && Ready condition available` | Mirror the Ready condition from infra machine | `InfrastructureReady=<mirrored status>; Reason=<mirrored reason>; Message=<mirrored message>; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine exists && Ready condition missing && status.initialization.infrastructureProvisioned=true` | Use fallback: infra is ready | `InfrastructureReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine exists && Ready condition missing && status.initialization.infrastructureProvisioned=false` | Use fallback: infra is not ready | `InfrastructureReady=False; Reason=NotReady; Message="<Kind> status.initialization.provisioned is false"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine exists && error parsing Ready condition` | No-op; invalid condition on infra machine | `InfrastructureReady=Unknown; Reason=InvalidConditionReported; Message="<error message>"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine not found && error occurred (not NotFound)` | No-op; internal error reading infra | `InfrastructureReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine not found && DeletionTimestamp set && infrastructureProvisioned=true` | No-op; infra was deleted during machine deletion | `InfrastructureReady=False; Reason=Deleted; Message="<Kind> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine not found && DeletionTimestamp set && infrastructureProvisioned=false` | No-op; infra never existed during deletion | `InfrastructureReady=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine not found && DeletionTimestamp not set && infrastructureProvisioned=true` | No-op; infra unexpectedly deleted while machine running | `InfrastructureReady=False; Reason=Deleted; Message="<Kind> has been deleted while the Machine still exists"; ObservedGeneration=metadata.generation` | `none` |
| `infraMachine not found && DeletionTimestamp not set && infrastructureProvisioned=false` | No-op; infra not yet created | `InfrastructureReady=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Condition Mirroring**: The primary mechanism for setting this condition is mirroring the `Ready` condition from the InfraMachine object. The `conditions.NewMirrorConditionFromUnstructured()` function handles the translation.

2. **Fallback Logic**: If the InfraMachine doesn't have a Ready condition (e.g., older providers), the controller uses `status.initialization.infrastructureProvisioned` as a fallback indicator.

3. **Infrastructure Providers**: Common infrastructure providers include AWSMachine, AzureMachine, GCPMachine, etc., which provision the actual compute resources.

4. **Deletion Handling**: Different messages are used depending on whether:
   - The infra was provisioned and then deleted (unexpected while running)
   - The infra never existed
   - The machine is being deleted

5. **Watch-Driven**: The reconciler watches both the Machine and the referenced InfraMachine objects for changes.

6. **Related Conditions**: 
   - `BootstrapConfigReady` indicates bootstrap status
   - `NodeHealthy` indicates node health
   - `Ready` is a summary that includes infrastructure readiness
   - `Available` indicates overall machine availability
