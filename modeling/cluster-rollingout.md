# Cluster Reconciler - RollingOut Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `RollingOut`

## Overview

The `RollingOut` condition on a Cluster reflects whether any rolling updates are in progress. It aggregates the `RollingOut` condition from the control plane (if reporting), MachineDeployments, and MachinePools. This condition has **negative polarity** (True = issue, False = healthy).

---

## Condition Transition Table — `RollingOut` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RollingOut`) | Next reconcile (Return) |
|---|---|---|---|
| `(cluster.Spec.ControlPlaneRef.IsDefined() && controlPlane == nil && !controlPlaneIsNotFound) || !getDescendantsSucceeded` | No-op; surface internal error | `RollingOut=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(aggregationSources) == 0` (no ControlPlane/MachineDeployments/MachinePools with RollingOut condition) | No-op; nothing to aggregate | `RollingOut=False; Reason=NotRollingOut; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all sources have RollingOut=False` | Aggregate conditions; no rollout in progress | `RollingOut=False; Reason=NotRollingOut; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any ControlPlane, MachineDeployment, or MachinePool has RollingOut=True` | Aggregate conditions; rollout in progress | `RollingOut=True; Reason=RollingOut; Message="<aggregated rollout details>"; ObservedGeneration=metadata.generation` | `none` |
| `any source has RollingOut=Unknown` | Aggregate conditions; surface unknown status | `RollingOut=Unknown; Reason=RollingOutUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `RollingOut=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Rolling out (issue state)
   - `False` = Not rolling out (healthy state)

2. **Aggregation Sources**:
   - Control plane object (only if it reports `RollingOut` condition; contract doesn't require it)
   - All `MachineDeployment` objects owned by the cluster
   - All `MachinePool` objects owned by the cluster

3. **Custom Merge Strategy**: Uses `DefaultMergeStrategy` with:
   - `TargetConditionHasPositivePolarity(false)` - accounts for negative polarity
   - Custom reason computation:
     - `RollingOut` for True status (issue)
     - `RollingOutUnknown` for Unknown status
     - `NotRollingOut` for False status (info)

4. **Control Plane Optional**: The control plane is only included if it reports the `RollingOut` condition. This means it won't surface as "Condition RollingOut not yet reported from...".

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
