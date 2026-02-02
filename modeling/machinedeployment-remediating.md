# MachineDeployment Reconciler - Remediating Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `Remediating`

## Overview

The `Remediating` condition on a MachineDeployment surfaces details about ongoing remediation of the controlled machines. It aggregates the `OwnerRemediated` condition from Machines that are unhealthy and need remediation by their owner (MachineSet). This condition has **negative polarity** (True = remediating, False = not remediating).

---

## Condition Transition Table — `Remediating` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Remediating`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) == 0` | No-op; no unhealthy machines | `Remediating=False; Reason=NotRemediating; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) > 0` | No-op; unhealthy machines exist but not being remediated by MD/MS | `Remediating=False; Reason=NotRemediating; Message="Machine(s) <names> are not healthy (not to be remediated by MachineDeployment/MachineSet)"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) > 0 && aggregation succeeds` | Aggregate OwnerRemediated conditions from machines | `Remediating=True; Reason=Remediating; Message="<aggregated remediation status from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) > 0 && aggregation fails` | Surface internal error | `Remediating=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Negative Polarity**: This condition has negative polarity:
   - `True` = Remediating (transitional state)
   - `False` = Not remediating (stable state)

2. **Machine Classification**:
   - **machinesToBeRemediated**: Machines filtered by `collections.IsUnhealthyAndOwnerRemediated` - these are machines that:
     - Have `HealthCheckSucceeded=False`
     - Have `OwnerRemediated` condition set to False (pending remediation by their owner)
   - **unhealthyMachines**: Machines filtered by `collections.IsUnhealthy` - these are machines that:
     - Have `HealthCheckSucceeded=False`
     - May or may not be remediated by the MachineDeployment/MachineSet (e.g., external remediation)

3. **Aggregation Logic**: When machines are being remediated (`len(machinesToBeRemediated) > 0`):
   - Uses `conditions.NewAggregateCondition` to aggregate `MachineOwnerRemediatedCondition` from machines
   - The aggregated message surfaces the remediation status from each machine
   - The reason is always `Remediating` (not computed from merge strategy)

4. **Unhealthy But Not Remediated**: When machines are unhealthy but not being remediated by MD/MS:
   - This can happen when machines are being remediated by external controllers (e.g., MachineHealthCheck with external remediation template)
   - The message lists the unhealthy machines and notes they won't be remediated by MachineDeployment/MachineSet

5. **Watch-Driven**: The reconciler is watch-driven; no `requeueAfter` is used for this condition. The controller reconciles when:
   - MachineDeployment changes
   - Owned MachineSets change
   - Controlled Machines change

6. **Relationship to MachineSet**: The actual remediation (machine deletion and recreation) is performed by the MachineSet controller, not the MachineDeployment controller. The MachineDeployment surfaces the aggregated remediation status from its MachineSets' machines.
