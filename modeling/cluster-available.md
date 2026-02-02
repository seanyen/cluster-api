# Cluster Reconciler - Available Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `Available`

## Overview

The `Available` condition on a Cluster is a summary condition computed from multiple sub-conditions. It reflects the overall health and readiness of the cluster, considering infrastructure, control plane, workers, remote connection, and topology reconciliation status.

---

## Condition Transition Table — `Available` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Available`) | Next reconcile (Return) |
|---|---|---|---|
| `deletionTimestamp.IsZero() && all sub-conditions healthy (Deleting=False, RemoteConnectionProbe=True, InfrastructureReady=True, ControlPlaneAvailable=True, WorkersAvailable=True, TopologyReconciled=True/missing)` | No-op; compute summary from sub-conditions | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && InfrastructureReady=False && reason=InfrastructureDoesNotExist (during provisioning)` | Compute summary; infrastructure not ready is surfaced | `Available=False; Reason=NotAvailable; Message="InfraCluster does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && ControlPlaneAvailable=False && reason=ControlPlaneDoesNotExist` | Compute summary; control plane not ready is surfaced | `Available=False; Reason=NotAvailable; Message="ControlPlane does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && RemoteConnectionProbe=False` | Compute summary; remote connection probe failure surfaced | `Available=False; Reason=NotAvailable; Message="Remote connection probe failed..."; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && WorkersAvailable=False` | Compute summary; workers not available surfaced | `Available=False; Reason=NotAvailable; Message="Workers not available..."; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && TopologyReconciled=False && reason=TopologyReconcileFailed` | Compute summary; topology failure surfaced as issue | `Available=False; Reason=NotAvailable; Message="Topology reconciliation failed"; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && TopologyReconciled=False && reason!=TopologyReconcileFailed (e.g., pending hooks)` | Compute summary; treated as info (not issue) | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `deletionTimestamp.IsZero() && any sub-condition=Unknown` | Compute summary; unknown status surfaced | `Available=Unknown; Reason=AvailableUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero() && Deleting=True` | Compute summary; deleting is treated with negative polarity | `Available=False; Reason=NotAvailable; Message="Cluster is deleting"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero() && InfrastructureReady=False && reason=InfrastructureDeleted` | Compute summary; during deletion, infrastructure deleted is treated as info | `Available=False; Reason=NotAvailable; Message="Deleting..."; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero() && ControlPlaneAvailable=False && reason=ControlPlaneDeleted` | Compute summary; during deletion, control plane deleted is treated as info | `Available=False; Reason=NotAvailable; Message="Deleting..."; ObservedGeneration=metadata.generation` | `none` |
| `internal error computing summary` | Set condition to Unknown | `Available=Unknown; Reason=AvailableInternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Summary Condition**: The `Available` condition is computed via `conditions.NewSummaryCondition()` which aggregates:
   - `Deleting` (negative polarity)
   - `RemoteConnectionProbe`
   - `InfrastructureReady`
   - `ControlPlaneAvailable`
   - `WorkersAvailable`
   - `TopologyReconciled` (if topology is defined)
   - Any custom availability gates from `spec.availabilityGates` or `ClusterClass.spec.availabilityGates`

2. **Custom Merge Strategy**: The controller uses a custom merge strategy (`clusterConditionCustomMergeStrategy`) that:
   - Treats `InfrastructureDeleted`, `ControlPlaneDeleted` as info priority during deletion
   - Treats `TopologyReconciled=False` as info except for `TopologyReconcileFailed` and `ClusterClassNotReconciled` reasons

3. **Watch-Driven**: The reconciler is watch-driven and does not use `requeueAfter` for the `Available` condition computation.
