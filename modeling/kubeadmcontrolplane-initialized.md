# KubeadmControlPlane Reconciler - Initialized Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `Initialized`

## Overview

The `Initialized` condition on a KubeadmControlPlane indicates whether the control plane has been initialized (kubeadm init completed successfully). This is determined by checking if the kubeadm-config ConfigMap exists in the workload cluster.

---

## Condition Transition Table — `Initialized` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Initialized`) | Next reconcile (Return) |
|---|---|---|---|
| `status.initialization.controlPlaneInitialized == true` | No-op; already initialized | `Initialized=True; Reason=Initialized; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `status.initialization.controlPlaneInitialized == false` | No-op; not yet initialized | `Initialized=False; Reason=NotInitialized; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Initialization Detection**: The control plane is considered initialized when:
   - The kubeadm-config ConfigMap exists in the workload cluster's kube-system namespace
   - This is a proxy for API Server being up and kubeadm init successfully completed

2. **One-Way Transition**: Once set to `True`, this condition does not change back to `False` even if the kubeadm-config ConfigMap is deleted.

3. **Protection During Remediation**: If there's only one machine and it's being remediated or deleted, the initialization check is skipped to prevent race conditions.

4. **Status Field Sync**: The `status.initialization.controlPlaneInitialized` field is synchronized with this condition.

5. **Watch-Driven**: The reconciler is watch-driven via the ClusterCache; no explicit `requeueAfter` is used for this condition.
