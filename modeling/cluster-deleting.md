# Cluster Reconciler - Deleting Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `Deleting`

## Overview

The `Deleting` condition on a Cluster reflects whether the cluster is being deleted and surfaces details about the ongoing deletion process. This condition has **negative polarity** (True = deleting, False = not deleting).

---

## Condition Transition Table — `Deleting` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Deleting`) | Next reconcile (Return) |
|---|---|---|---|
| `cluster.DeletionTimestamp.IsZero()` | No-op; cluster is not being deleted | `Deleting=False; Reason=NotDeleting; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!cluster.DeletionTimestamp.IsZero() && deletingReason != ""` | Surface deletion progress from reconciler | `Deleting=True; Reason=<deletingReason>; Message="<deletingMessage>"; ObservedGeneration=metadata.generation` | `requeue` or `none` (depends on deletion phase) |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Cluster is being deleted (transitional state)
   - `False` = Cluster is not being deleted (stable state)

2. **Deletion Reason and Message**: When the cluster is being deleted, the `deletingReason` and `deletingMessage` are computed by the reconciler's deletion logic, which tracks:
   - Deletion of owned resources (MachineDeployments, MachineSets, Machines, MachinePools)
   - Deletion of infrastructure cluster
   - Deletion of control plane
   - Waiting for external objects

3. **Typical Reasons During Deletion**:
   - `DeletingMachines` - Waiting for machines to be deleted
   - `DeletingInfrastructure` - Waiting for infrastructure cluster to be deleted
   - `DeletingControlPlane` - Waiting for control plane to be deleted
   - `DeletingExternalObjects` - Waiting for external objects to be deleted

4. **Impact on Available**: The `Deleting=True` condition is included in the `Available` condition summary with negative polarity, causing `Available` to become False when the cluster is deleting.

5. **Watch-Driven**: The reconciler is watch-driven. During deletion, it may requeue to poll for deletion completion of owned resources.
