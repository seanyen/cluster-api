# Machine Reconciler - Deleting Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `Deleting`

## Overview

The `Deleting` condition on a Machine indicates whether the Machine is being deleted. This condition tracks the deletion progress through various stages (e.g., waiting for node drain, waiting for volume detach) and helps operators understand where the deletion process currently stands.

---

## Condition Transition Table — `Deleting` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Deleting`) | Next reconcile (Return) |
|---|---|---|---|
| `DeletionTimestamp.IsZero()` | No-op; not deleting | `Deleting=False; Reason=NotDeleting; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=WaitingForPreDrainHook` | Execute pre-drain hooks | `Deleting=True; Reason=WaitingForPreDrainHook; Message="<hook details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=DrainingNode` | Continue draining node | `Deleting=True; Reason=DrainingNode; Message="<drain progress details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=WaitingForVolumeDetach` | Wait for volumes to detach | `Deleting=True; Reason=WaitingForVolumeDetach; Message="<volume details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=WaitingForPreTerminateHook` | Execute pre-terminate hooks | `Deleting=True; Reason=WaitingForPreTerminateHook; Message="<hook details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=DeletingInfrastructure` | Delete infrastructure | `Deleting=True; Reason=DeletingInfrastructure; Message="<infra deletion details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=DeletingBootstrapConfig` | Delete bootstrap config | `Deleting=True; Reason=DeletingBootstrapConfig; Message="<bootstrap deletion details>"; ObservedGeneration=metadata.generation` | `none` |
| `!DeletionTimestamp.IsZero() && deletingReason=DeletingNode` | Delete the Kubernetes Node | `Deleting=True; Reason=DeletingNode; Message="<node deletion details>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Deletion Stages**: Machine deletion proceeds through multiple stages:
   - Pre-drain hooks (if configured)
   - Node drain (cordon, evict pods)
   - Volume detach wait
   - Pre-terminate hooks (if configured)
   - Infrastructure deletion
   - Bootstrap config deletion
   - Node deletion

2. **Reason Tracks Stage**: The `Reason` field indicates which stage of deletion the machine is currently in.

3. **Message Contains Details**: The `Message` field contains details about the current deletion stage, such as:
   - Which pods are being evicted
   - Which volumes are being detached
   - Which hooks are being executed

4. **Negative Polarity**: The `Deleting` condition uses negative polarity in the `Ready` condition summary - when `Deleting=True`, the machine is considered not ready.

5. **Duration Tracking**: The Machine tracks deletion timing in `status.deletion` fields like `nodeDrainStartTime` to help identify stuck deletions.

6. **Watch-Driven**: The reconciler is watch-driven, reacting to changes in the Machine, Node, and referenced objects.

7. **Related Conditions**: 
   - `Ready` is False when Deleting=True
   - `Available` is False when Deleting=True
   - `NodeHealthy`/`NodeReady` track node state during deletion
