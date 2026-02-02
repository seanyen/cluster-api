# MachineSet Reconciler - MachinesReady Condition

**Reconciler**: `internal/controllers/machineset/machineset_controller.go`  
**Primary Reconciled Object**: `MachineSet`  
**Condition Type Modeled**: `MachinesReady`

## Overview

The `MachinesReady` condition on a MachineSet is an aggregate condition that reflects the readiness of all Machines owned by the MachineSet. It is computed by aggregating the `Ready` condition from each owned Machine.

---

## Condition Transition Table — `MachinesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `getAndAdoptMachinesForMachineSetSucceeded == false` | No-op; cannot determine machine status | `MachinesReady=Unknown; Reason=MachinesReadyInternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) == 0` | No-op; no replicas to check | `MachinesReady=True; Reason=MachinesReadyNoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && all machines have Ready=True` | Aggregate Machine Ready conditions | `MachinesReady=True; Reason=MachinesReady; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && some machines have Ready=False` | Aggregate Machine Ready conditions | `MachinesReady=False; Reason=MachinesNotReady; Message="<aggregated issues from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && some machines have Ready=Unknown && none have Ready=False` | Aggregate Machine Ready conditions | `MachinesReady=Unknown; Reason=MachinesReadyUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `error aggregating Machine Ready conditions` | Set condition to Unknown | `MachinesReady=Unknown; Reason=MachinesReadyInternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Aggregate Condition**: The `MachinesReady` condition is computed using `conditions.NewAggregateCondition()` which:
   - Iterates through all owned Machines
   - Aggregates their `Ready` condition status
   - Uses a custom merge strategy to compute the final reason

2. **In-Place Updates**: If a Machine is undergoing an in-place update (`inplace.IsUpdateInProgress(machine)`), it is not counted toward `readyReplicas`, `availableReplicas`, or `upToDateReplicas`.

3. **Watch-Driven**: The reconciler is watch-driven via watches on owned Machines.

4. **Relationship with Replica Counters**: The `status.readyReplicas` counter is updated based on Machines that have `Ready=True` and are not in-place updating.
