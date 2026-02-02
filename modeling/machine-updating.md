# Machine Reconciler - Updating Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `Updating`

## Overview

The `Updating` condition on a Machine indicates whether an in-place update is currently in progress on the Machine. In-place updates allow certain changes to be applied without replacing the Machine, such as updating Kubernetes version in some scenarios.

---

## Condition Transition Table — `Updating` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Updating`) | Next reconcile (Return) |
|---|---|---|---|
| `inplace.IsUpdateInProgress(machine) == true && updatingReason provided` | Continue in-place update | `Updating=True; Reason=<updatingReason>; Message=<updatingMessage>; ObservedGeneration=metadata.generation` | `none` |
| `inplace.IsUpdateInProgress(machine) == true && no updatingReason provided` | Continue in-place update | `Updating=True; Reason=InPlaceUpdating; Message="In-place update in progress"; ObservedGeneration=metadata.generation` | `none` |
| `inplace.IsUpdateInProgress(machine) == false` | No-op; not updating | `Updating=False; Reason=NotUpdating; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **In-Place Updates**: Not all changes require machine replacement. Some control plane implementations support in-place updates for:
   - Kubernetes version upgrades (cordon, upgrade, uncordon)
   - Certificate rotations
   - Configuration changes that don't require reprovisioning

2. **Update Detection**: The `inplace.IsUpdateInProgress()` function checks for annotations or status fields that indicate an in-place update is ongoing.

3. **Negative Polarity**: The `Updating` condition uses negative polarity in the `Ready` condition summary - when `Updating=True`, the machine is considered not ready.

4. **Coordination with Rollout**: The MachineDeployment and MachineSet controllers coordinate with this condition to:
   - Not create new machines while an update is in progress
   - Not delete updating machines
   - Properly sequence updates across the deployment

5. **Reason Customization**: Different update types can provide custom reasons and messages to better describe what type of update is in progress.

6. **Watch-Driven**: The reconciler is watch-driven, reacting to changes in the Machine and its annotations/status.

7. **Related Conditions**: 
   - `Ready` is False when Updating=True
   - `UpToDate` may be False while updating to new spec
   - `Available` may be affected during updates
