# Cluster Reconciler - InfrastructureReady Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `InfrastructureReady`

## Overview

The `InfrastructureReady` condition on a Cluster indicates whether the infrastructure (InfraCluster) for the cluster has been successfully provisioned. This condition is primarily mirrored from the infrastructure provider's cluster object.

---

## Condition Transition Table — `InfrastructureReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `InfrastructureReady`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.infrastructureRef not defined && spec.topology defined && DeletionTimestamp.IsZero()` | No-op; waiting for ClusterClass topology reconciliation | `InfrastructureReady=False; Reason=DoesNotExist; Message="Waiting for cluster topology to be reconciled"; ObservedGeneration=metadata.generation` | `none` |
| `spec.infrastructureRef not defined && spec.topology defined && !DeletionTimestamp.IsZero()` | No-op; deleting, infra not relevant | `InfrastructureReady=False; Reason=DoesNotExist; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster exists && error parsing Ready condition` | No-op; invalid condition on infra cluster | `InfrastructureReady=Unknown; Reason=InvalidConditionReported; Message="<error message>"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster exists && Ready condition available` | Mirror the Ready condition from infra cluster | `InfrastructureReady=<mirrored status>; Reason=<mirrored reason>; Message=<mirrored message>; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster exists && Ready condition missing && status.initialization.infrastructureProvisioned=true` | Use fallback: infra is ready | `InfrastructureReady=True; Reason=Ready; Message="<Kind> is ready (fallback)"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster exists && Ready condition missing && status.initialization.infrastructureProvisioned=false` | Use fallback: infra is not ready | `InfrastructureReady=False; Reason=NotReady; Message="Waiting for <Kind> to report ready (fallback)"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster not found && error occurred (not NotFound)` | No-op; internal error reading infra | `InfrastructureReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster not found && DeletionTimestamp set && infrastructureProvisioned=true` | No-op; infra was deleted during cluster deletion | `InfrastructureReady=False; Reason=Deleted; Message="<Kind> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster not found && DeletionTimestamp set && infrastructureProvisioned=false` | No-op; infra never existed during deletion | `InfrastructureReady=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster not found && DeletionTimestamp not set && infrastructureProvisioned=true` | No-op; infra unexpectedly deleted while cluster running | `InfrastructureReady=False; Reason=Deleted; Message="<Kind> has been deleted while the cluster still exists"; ObservedGeneration=metadata.generation` | `none` |
| `infraCluster not found && DeletionTimestamp not set && infrastructureProvisioned=false` | No-op; infra not yet created | `InfrastructureReady=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Condition Mirroring**: The primary mechanism for setting this condition is mirroring the `Ready` condition from the InfraCluster object. The `conditions.NewMirrorConditionFromUnstructured()` function handles the translation.

2. **Fallback Logic**: If the InfraCluster doesn't have a Ready condition (e.g., older providers), the controller uses `status.initialization.infrastructureProvisioned` as a fallback indicator.

3. **ClusterClass Support**: For clusters using ClusterClass (`spec.topology` is defined), the `infrastructureRef` may not be set until the topology reconciler creates the infrastructure object.

4. **Deletion Handling**: Different messages are used depending on whether:
   - The infra was provisioned and then deleted (unexpected)
   - The infra never existed
   - The cluster is being deleted

5. **Watch-Driven**: The reconciler watches both the Cluster and the referenced InfraCluster objects for changes.

6. **Related Conditions**: 
   - `ControlPlaneAvailable` indicates the control plane status
   - `Available` is a summary that includes infrastructure readiness
