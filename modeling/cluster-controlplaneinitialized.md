# Cluster Reconciler - ControlPlaneInitialized Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ControlPlaneInitialized`

## Overview

The `ControlPlaneInitialized` condition on a Cluster indicates whether the control plane has been successfully initialized. This is a one-time condition that becomes True once the control plane is initialized and remains True thereafter. It's used to gate operations that require a functioning API server.

---

## Condition Transition Table — `ControlPlaneInitialized` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneInitialized`) | Next reconcile (Return) |
|---|---|---|---|
| `ControlPlaneInitialized condition already True` | No-op; initialization is a one-way transition | No change (condition remains True) | `none` |
| `spec.controlPlaneRef not defined && spec.topology defined && DeletionTimestamp.IsZero()` | No-op; waiting for ClusterClass topology reconciliation | `ControlPlaneInitialized=Unknown; Reason=DoesNotExist; Message="Waiting for cluster topology to be reconciled"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef not defined && spec.topology defined && !DeletionTimestamp.IsZero()` | No-op; deleting | `ControlPlaneInitialized=Unknown; Reason=DoesNotExist; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef defined && controlPlane not found && error occurred (not NotFound)` | No-op; internal error reading control plane | `ControlPlaneInitialized=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef defined && controlPlane not found (NotFound)` | No-op; control plane object not created yet | `ControlPlaneInitialized=Unknown; Reason=DoesNotExist; Message="<Kind> does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef defined && controlPlane exists && status.initialized == true` | Mark initialized | `ControlPlaneInitialized=True; Reason=Initialized; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef defined && controlPlane exists && status.initialized == false` | No-op; waiting for initialization | `ControlPlaneInitialized=False; Reason=NotInitialized; Message="Control plane not yet initialized"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef defined && controlPlane exists && error reading status.initialized (not field not found)` | No-op; internal error | `ControlPlaneInitialized=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef not defined && standalone machines && getDescendantsSucceeded == false` | No-op; internal error listing machines | `ControlPlaneInitialized=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef not defined && standalone machines && any machine has nodeRef` | Mark initialized (at least one CP machine has a node) | `ControlPlaneInitialized=True; Reason=Initialized; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `spec.controlPlaneRef not defined && standalone machines && no machine has nodeRef` | No-op; waiting for first node | `ControlPlaneInitialized=False; Reason=NotInitialized; Message="Waiting for the first control plane machine to have status.nodeRef set"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **One-Way Transition**: Once the `ControlPlaneInitialized` condition becomes True, it stays True. This is intentional because many operations gate on this condition, and reverting it would cause issues.

2. **Control Plane Object vs Standalone Machines**: The cluster can use either:
   - A control plane object (e.g., KubeadmControlPlane) which reports `status.initialized`
   - Standalone control plane machines, where initialization is determined by whether any machine has a node

3. **ClusterClass Support**: For clusters using ClusterClass, the `controlPlaneRef` may not be set until the topology reconciler creates the control plane object.

4. **Gate for Other Operations**: This condition is used as a gate for:
   - Node health checking (requires API server connection)
   - Worker node provisioning
   - Various status updates that require cluster connectivity

5. **Watch-Driven**: The reconciler watches the Cluster, control plane object, and control plane machines.

6. **Related Conditions**: 
   - `ControlPlaneAvailable` indicates ongoing availability (can fluctuate)
   - `InfrastructureReady` must be True before initialization can complete
   - `Available` is a summary that depends on initialization
