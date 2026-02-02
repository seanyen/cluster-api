# Cluster Reconciler - ControlPlaneAvailable Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ControlPlaneAvailable`

## Overview

The `ControlPlaneAvailable` condition on a Cluster indicates whether the control plane is available and serving the Kubernetes API. This condition is primarily mirrored from the control plane provider's object (e.g., KubeadmControlPlane).

---

## Condition Transition Table — `ControlPlaneAvailable` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneAvailable`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.controlPlaneRef not defined && spec.topology defined && DeletionTimestamp.IsZero()` | No-op; waiting for ClusterClass topology reconciliation | `ControlPlaneAvailable=False; Reason=DoesNotExist; Message="Waiting for cluster topology to be reconciled"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef not defined && spec.topology defined && !DeletionTimestamp.IsZero()` | No-op; deleting, control plane not relevant | `ControlPlaneAvailable=False; Reason=DoesNotExist; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane exists && error parsing Available condition` | No-op; invalid condition on control plane | `ControlPlaneAvailable=Unknown; Reason=InvalidConditionReported; Message="<error message>"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane exists && Available condition available` | Mirror the Available condition from control plane | `ControlPlaneAvailable=<mirrored status>; Reason=<mirrored reason>; Message=<mirrored message>; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane exists && Available condition missing && status.initialization.controlPlaneInitialized=true` | Use fallback: control plane is available | `ControlPlaneAvailable=True; Reason=Available; Message="<Kind> is available (fallback)"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane exists && Available condition missing && status.initialization.controlPlaneInitialized=false` | Use fallback: control plane is not available | `ControlPlaneAvailable=False; Reason=NotAvailable; Message="Waiting for <Kind> to report available (fallback)"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane not found && error occurred (not NotFound)` | No-op; internal error reading control plane | `ControlPlaneAvailable=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane not found && DeletionTimestamp set && controlPlaneInitialized=true` | No-op; control plane was deleted during cluster deletion | `ControlPlaneAvailable=False; Reason=Deleted; Message="<Kind> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane not found && DeletionTimestamp set && controlPlaneInitialized=false` | No-op; control plane never existed during deletion | `ControlPlaneAvailable=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane not found && DeletionTimestamp not set && controlPlaneInitialized=true` | No-op; control plane unexpectedly deleted while cluster running | `ControlPlaneAvailable=False; Reason=Deleted; Message="<Kind> has been deleted while the cluster still exists"; ObservedGeneration=metadata.generation` | `none` |
| `controlPlane not found && DeletionTimestamp not set && controlPlaneInitialized=false` | No-op; control plane not yet created | `ControlPlaneAvailable=False; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Condition Mirroring**: The primary mechanism for setting this condition is mirroring the `Available` condition from the control plane object. The `conditions.NewMirrorConditionFromUnstructured()` function handles the translation.

2. **Fallback Logic**: If the control plane doesn't have an Available condition (e.g., older providers), the controller uses `status.initialization.controlPlaneInitialized` as a fallback indicator.

3. **ClusterClass Support**: For clusters using ClusterClass (`spec.topology` is defined), the `controlPlaneRef` may not be set until the topology reconciler creates the control plane object.

4. **Initialization vs Availability**: Note that `controlPlaneInitialized` is used as the fallback, which indicates the control plane was initialized at some point. The Available condition specifically tracks ongoing availability.

5. **Deletion Handling**: Different messages are used depending on whether:
   - The control plane was initialized and then deleted (unexpected)
   - The control plane never existed
   - The cluster is being deleted

6. **Watch-Driven**: The reconciler watches both the Cluster and the referenced control plane objects for changes.

7. **Related Conditions**: 
   - `InfrastructureReady` indicates infrastructure status
   - `ControlPlaneInitialized` indicates one-time initialization
   - `Available` is a summary that includes control plane availability
