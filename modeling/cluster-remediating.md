# Cluster Reconciler - Remediating Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `Remediating`

## Overview

The `Remediating` condition on a Cluster reflects whether any machine remediation is in progress. It aggregates the `OwnerRemediated` condition from Machines that are marked for remediation. This condition has **negative polarity** (True = remediating, False = not remediating).

---

## Condition Transition Table — `Remediating` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Remediating`) | Next reconcile (Return) |
|---|---|---|---|
| `!getMachinesSucceeded` (error listing machines) | No-op; surface internal error | `Remediating=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
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
   - `machinesToBeRemediated`: Machines that have been marked for remediation by MachineHealthCheck
   - `unhealthyMachines`: Machines that are unhealthy but not being remediated (e.g., remediation blocked by maxUnhealthy)

3. **Aggregation Source**: When remediation is in progress, aggregates `OwnerRemediated` conditions from `machinesToBeRemediated`.

4. **MachinePool Machine Exclusion**: MachinePool machines (identified by `cluster.x-k8s.io/pool-name` label) are excluded from the aggregation.

5. **Unhealthy Machine Reporting**: When there are unhealthy machines that are not being remediated, they are listed in the condition message (up to 3 machine names, then truncated).

6. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition.
