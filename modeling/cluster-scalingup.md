# Cluster Reconciler - ScalingUp Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `ScalingUp`

## Overview

The `ScalingUp` condition on a Cluster reflects whether any scale-up operations are in progress. It aggregates the `ScalingUp` condition from the control plane (if reporting), MachineDeployments, MachinePools, and standalone MachineSets. This condition has **negative polarity** (True = scaling up, False = stable).

---

## Condition Transition Table — `ScalingUp` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ScalingUp`) | Next reconcile (Return) |
|---|---|---|---|
| `(cluster.Spec.ControlPlaneRef.IsDefined() && controlPlane == nil && !controlPlaneIsNotFound) || !getDescendantsSucceeded` | No-op; surface internal error | `ScalingUp=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(aggregationSources) == 0` (no ControlPlane/MachineDeployments/MachinePools/MachineSets with ScalingUp condition) | No-op; nothing to aggregate | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all sources have ScalingUp=False` | Aggregate conditions; no scale-up in progress | `ScalingUp=False; Reason=NotScalingUp; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any ControlPlane, MachineDeployment, MachinePool, or MachineSet has ScalingUp=True` | Aggregate conditions; scale-up in progress | `ScalingUp=True; Reason=ScalingUp; Message="<aggregated scaling details>"; ObservedGeneration=metadata.generation` | `none` |
| `any source has ScalingUp=Unknown` | Aggregate conditions; surface unknown status | `ScalingUp=Unknown; Reason=ScalingUpUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `ScalingUp=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Scaling up (transitional state)
   - `False` = Not scaling up (stable state)

2. **Aggregation Sources**:
   - Control plane object (only if it reports `ScalingUp` condition; contract doesn't require it)
   - All `MachineDeployment` objects owned by the cluster
   - All `MachinePool` objects owned by the cluster
   - Standalone `MachineSet` objects directly owned by the cluster (not owned by MachineDeployments)

3. **Standalone MachineSet Filtering**: Only MachineSets that are directly owned by the Cluster (via `util.IsOwnedByObject`) are included, not those owned by MachineDeployments.

4. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with:
   - `TargetConditionHasPositivePolarity(false)` - accounts for negative polarity
   - Custom reason computation:
     - `ScalingUp` for True status (issue)
     - `ScalingUpUnknown` for Unknown status
     - `NotScalingUp` for False status (info)

5. **Control Plane Optional**: The control plane is only included if it reports the `ScalingUp` condition.

6. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
