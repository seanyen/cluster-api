# Machine Reconciler - UpToDate Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `UpToDate`

## Overview

The `UpToDate` condition on a Machine indicates whether the Machine's configuration matches the desired state from its owning MachineDeployment. This condition is only set on Machines owned by a MachineSet that is owned by a MachineDeployment; it is not set on standalone machines or machines controlled by other owners (like KubeadmControlPlane, which sets its own UpToDate condition).

---

## Condition Transition Table — `UpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `UpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `owningMachineSet == nil || owningMachineDeployment == nil` (standalone Machine or MachineSet not owned by MD) | No-op; do not set condition | (condition not set by Machine controller) | `none` |
| `machineSet.spec.template != machineDeployment.spec.template` (template drift) | No-op; surface drift reasons | `UpToDate=False; Reason=NotUpToDate; Message="* <drift details>"; ObservedGeneration=metadata.generation` | `none` |
| `machineDeployment.spec.rollout.after expired && machineSet.creationTimestamp <= rollout.after` | No-op; surface rollout expired | `UpToDate=False; Reason=NotUpToDate; Message="* MachineDeployment spec.rolloutAfter expired"; ObservedGeneration=metadata.generation` | `none` |
| `machine.annotations[in-place-update-in-progress] == "true"` (in-place update in progress) | No-op; surface in-place update | `UpToDate=False; Reason=Updating; Message="* In-place update in progress"; ObservedGeneration=metadata.generation` | `none` |
| `machineSet.spec.template == machineDeployment.spec.template && no in-place update` | No-op; machine is up to date | `UpToDate=True; Reason=UpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Condition Ownership**: The Machine controller sets this condition only for Machines owned by a MachineDeployment (via MachineSet). Other owners (KubeadmControlPlane, etc.) manage their own UpToDate conditions on their Machines.

2. **Template Comparison**: The comparison is done between:
   - Current state: `machineSet.spec.template` (not the Machine itself, for consistency with MachineDeployment controller)
   - Desired state: `machineDeployment.spec.template`

3. **Drift Detection**: Detected drifts include:
   - Kubernetes version changes
   - InfrastructureRef template changes
   - Bootstrap config template changes
   - Metadata label/annotation changes
   - Spec field changes (excluding in-place mutable fields)

4. **In-Place Updates**: When an in-place update is in progress (annotation present), the condition is False with reason `Updating`.

5. **Impact on RollingOut**: The `UpToDate=False` condition on Machines is aggregated by MachineDeployment to compute its `RollingOut` condition.

6. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
