# Cluster Reconciler - ScalingDown Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ScalingDown`

## Overview

The `ScalingDown` condition on a Cluster reflects whether any scale-down operations are in progress. It aggregates the `ScalingDown` condition from the control plane (if reporting), MachineDeployments, MachinePools, and standalone MachineSets. This condition has **negative polarity** (True = scaling down, False = stable).

---

## Condition Transition Table — `ScalingDown` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingDown`) | Next reconcile (Return) |
|---|---|---|---|
| `(cluster.Spec.ControlPlaneRef.IsDefined() && controlPlane == nil && !controlPlaneIsNotFound) || !getDescendantsSucceeded` | No-op; surface internal error | `ScalingDown=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(aggregationSources) == 0` (no ControlPlane/MachineDeployments/MachinePools/MachineSets with ScalingDown condition) | No-op; nothing to aggregate | `ScalingDown=False; Reason=NotScalingDown; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all sources have ScalingDown=False` | Aggregate conditions; no scale-down in progress | `ScalingDown=False; Reason=NotScalingDown; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any ControlPlane, MachineDeployment, MachinePool, or MachineSet has ScalingDown=True` | Aggregate conditions; scale-down in progress | `ScalingDown=True; Reason=ScalingDown; Message="<aggregated scaling details>"; ObservedGeneration=metadata.generation` | `none` |
| `any source has ScalingDown=Unknown` | Aggregate conditions; surface unknown status | `ScalingDown=Unknown; Reason=ScalingDownUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `ScalingDown=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Scaling down (transitional state)
   - `False` = Not scaling down (stable state)

2. **Aggregation Sources**:
   - Control plane object (only if it reports `ScalingDown` condition; contract doesn't require it)
   - All `MachineDeployment` objects owned by the cluster
   - All `MachinePool` objects owned by the cluster
   - Standalone `MachineSet` objects directly owned by the cluster (not owned by MachineDeployments)

3. **Standalone MachineSet Filtering**: Only MachineSets that are directly owned by the Cluster (via `util.IsOwnedByObject`) are included, not those owned by MachineDeployments.

4. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with:
   - `TargetConditionHasPositivePolarity(false)` - accounts for negative polarity
   - Custom reason computation:
     - `ScalingDown` for True status (issue)
     - `ScalingDownUnknown` for Unknown status
     - `NotScalingDown` for False status (info)

5. **Control Plane Optional**: The control plane is only included if it reports the `ScalingDown` condition.

6. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
