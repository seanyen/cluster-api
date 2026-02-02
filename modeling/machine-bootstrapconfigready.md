# Machine Reconciler - BootstrapConfigReady Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `BootstrapConfigReady`

## Overview

The `BootstrapConfigReady` condition on a Machine indicates whether the bootstrap configuration (e.g., KubeadmConfig) is ready and has produced the bootstrap data secret. This condition is primarily mirrored from the bootstrap provider's object.

---

## Condition Transition Table — `BootstrapConfigReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `BootstrapConfigReady`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.bootstrap.configRef not defined` | No-op; dataSecretName provided directly | `BootstrapConfigReady=True; Reason=DataSecretProvided; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig exists && Ready condition available` | Mirror the Ready condition from bootstrap config | `BootstrapConfigReady=<mirrored status>; Reason=<mirrored reason>; Message=<mirrored message>; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig exists && Ready condition missing && status.initialization.bootstrapDataSecretCreated=true` | Use fallback: bootstrap is ready | `BootstrapConfigReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig exists && Ready condition missing && status.initialization.bootstrapDataSecretCreated=false` | Use fallback: bootstrap is not ready | `BootstrapConfigReady=False; Reason=NotReady; Message="<Kind> status.initialization.dataSecretCreated is false"; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig exists && error parsing Ready condition` | No-op; invalid condition on bootstrap config | `BootstrapConfigReady=Unknown; Reason=InvalidConditionReported; Message="<error message>"; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig not found && error occurred (not NotFound)` | No-op; internal error reading bootstrap | `BootstrapConfigReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig not found && DeletionTimestamp set && bootstrapDataSecretCreated=true` | No-op; bootstrap was deleted during machine deletion | `BootstrapConfigReady=False; Reason=Deleted; Message="<Kind> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `bootstrapConfig not found && (DeletionTimestamp not set OR bootstrapDataSecretCreated=false)` | No-op; bootstrap config not yet created | `BootstrapConfigReady=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Direct Data Secret**: If `spec.bootstrap.dataSecretName` is provided directly without a `configRef`, the condition is immediately set to True with reason `DataSecretProvided`.

2. **Condition Mirroring**: The primary mechanism for setting this condition is mirroring the `Ready` condition from the BootstrapConfig object. The `conditions.NewMirrorConditionFromUnstructured()` function handles the translation.

3. **Fallback Logic**: If the BootstrapConfig doesn't have a Ready condition (e.g., older providers), the controller uses `status.initialization.bootstrapDataSecretCreated` as a fallback indicator.

4. **Bootstrap Providers**: Common bootstrap providers include KubeadmConfig, which produces cloud-init or ignition data for machine initialization.

5. **Watch-Driven**: The reconciler watches both the Machine and the referenced BootstrapConfig objects for changes.

6. **Related Conditions**: 
   - `InfrastructureReady` indicates infrastructure status
   - `Ready` is a summary that includes bootstrap readiness
   - `Available` indicates overall machine availability
