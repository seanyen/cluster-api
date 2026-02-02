# KubeadmControlPlane Reconciler - Deleting Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `Deleting`

## Overview

The `Deleting` condition on a KubeadmControlPlane indicates whether the control plane is being deleted and surfaces details about the ongoing deletion process. This condition has **negative polarity** (True = deleting, False = not deleting).

---

## Condition Transition Table — `Deleting` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Deleting`) | Next reconcile (Return) |
|---|---|---|---|
| `deletionTimestamp.IsZero()` | No-op; not being deleted | `Deleting=False; Reason=NotDeleting; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero() && deletingReason != ""` | Surface deletion progress | `Deleting=True; Reason=<deletingReason>; Message="<deletingMessage>"; ObservedGeneration=metadata.generation` | `requeue` or `none` (depends on deletion phase) |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Control plane is being deleted (transitional state)
   - `False` = Control plane is not being deleted (stable state)

2. **Deletion Reason and Message**: When the control plane is being deleted, the `deletingReason` and `deletingMessage` are computed by the reconciler's deletion logic, which tracks:
   - Deletion of owned Machines
   - Removal of etcd members
   - Cleanup of infrastructure machine templates
   - Waiting for external dependencies

3. **Typical Reasons During Deletion**:
   - `DeletingMachines` - Waiting for machines to be deleted
   - `DeletingEtcdMembers` - Removing etcd members from the cluster
   - `WaitingForMachineDeletion` - Waiting for machine finalizers to complete

4. **Control Plane Deletion Order**: KCP ensures orderly deletion:
   - Machines are deleted one at a time
   - Etcd members are removed before machine deletion
   - The last machine deletion is handled specially

5. **Watch-Driven**: The reconciler is watch-driven. During deletion, it may requeue to poll for deletion completion of owned resources.
