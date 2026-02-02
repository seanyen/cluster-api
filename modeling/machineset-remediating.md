# MachineSet Reconciler - Remediating Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `Remediating`

## Overview

The `Remediating` condition on a MachineSet indicates whether any of its owned Machines are currently being remediated (e.g., by a MachineHealthCheck). This condition aggregates the `OwnerRemediated` condition from Machines that need remediation.

---

## Condition Transition Table — `Remediating` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Remediating`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachinesForMachineSetSucceeded == false` | No-op; internal error listing machines | `Remediating=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) == 0` | No-op; no machines need remediation | `Remediating=False; Reason=NotRemediating; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) == 0 && len(unhealthyMachines) > 0` | No-op; machines are unhealthy but not yet being remediated | `Remediating=False; Reason=NotRemediating; Message="<aggregated unhealthy machine info>"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) > 0 && aggregation succeeds` | Track remediation progress | `Remediating=True; Reason=Remediating; Message="<aggregated OwnerRemediated condition messages>"; ObservedGeneration=metadata.generation` | `none` |
| `len(machinesToBeRemediated) > 0 && aggregation fails` | No-op; internal error aggregating conditions | `Remediating=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Machines To Be Remediated**: These are Machines that have the `OwnerRemediated` condition set, indicating a MachineHealthCheck has marked them for remediation.

2. **Unhealthy Machines**: These are Machines that are unhealthy but may not yet be marked for remediation. The condition message surfaces this information even when not actively remediating.

3. **Condition Aggregation**: The `Remediating` condition aggregates the `OwnerRemediated` conditions from all machines being remediated using `conditions.NewAggregateCondition`.

4. **MachineHealthCheck Integration**: This condition works in conjunction with MachineHealthCheck, which sets the `OwnerRemediated` condition on unhealthy Machines.

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineSet and its owned Machines.

6. **Related Conditions**: 
   - `MachinesReady` indicates whether owned Machines are ready
   - Machine's `OwnerRemediated` condition is aggregated into this condition
