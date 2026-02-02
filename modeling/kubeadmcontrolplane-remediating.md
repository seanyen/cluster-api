# KubeadmControlPlane Reconciler - Remediating Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `Remediating`

## Overview

The `Remediating` condition on a KubeadmControlPlane indicates whether any machine remediation is in progress. It aggregates the `OwnerRemediated` condition from Machines that are marked for remediation. This condition has **negative polarity** (True = remediating, False = not remediating).

---

## Condition Transition Table — `Remediating` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Remediating`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) == 0` | No-op; no remediation in progress, no unhealthy machines | `Remediating=False; Reason=NotRemediating; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) > 0` | No-op; surface unhealthy machines not being remediated | `Remediating=False; Reason=NotRemediating; Message="Machine(s) <names> are not healthy (not to be remediated)"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) > 0` | Aggregate OwnerRemediated conditions from machines being remediated | `Remediating=True; Reason=Remediating; Message="<aggregated remediation details>"; ObservedGeneration=metadata.generation` | `none` |
| `internal error during aggregation` | Set condition to Unknown | `Remediating=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Remediation in progress (transitional state)
   - `False` = Not remediating (stable state)

2. **Machine Categories**:
   - `machinesToBeRemediated`: Machines owned by KCP that have been marked for remediation
   - `unhealthyMachines`: Machines that are unhealthy but not being remediated (e.g., remediation blocked by minimum healthy count)

3. **Aggregation Source**: When remediation is in progress, aggregates `OwnerRemediated` conditions from `machinesToBeRemediated`.

4. **Unhealthy Machine Reporting**: When there are unhealthy machines that are not being remediated, they are listed in the condition message.

5. **KCP Remediation Logic**: KCP has its own remediation logic separate from MachineHealthCheck:
   - Respects etcd quorum requirements
   - Ensures minimum number of healthy control plane nodes
   - Handles remediation retries with backoff

6. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
