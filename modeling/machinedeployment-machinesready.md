# MachineDeployment Reconciler - MachinesReady Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `MachinesReady`

## Overview

The `MachinesReady` condition on a MachineDeployment is an aggregate condition that summarizes the `Ready` condition from all Machines owned by the MachineDeployment (through its MachineSets). This provides a single view of whether all machines in the deployment are ready.

---

## Condition Transition Table — `MachinesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `MachinesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `len(machines) == 0` | No-op; no machines to aggregate | `MachinesReady=True; Reason=NoReplicas; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && aggregation fails` | No-op; internal error aggregating conditions | `MachinesReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && all machines Ready=True` | Aggregate machine Ready conditions | `MachinesReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && some machines Ready=False` | Aggregate machine Ready conditions | `MachinesReady=False; Reason=NotReady; Message="<aggregated messages from machines>"; ObservedGeneration=metadata.generation` | `none` |
| `len(machines) > 0 && some machines Ready=Unknown && none False` | Aggregate machine Ready conditions | `MachinesReady=Unknown; Reason=Unknown; Message="<aggregated messages from machines>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Condition Aggregation**: The `MachinesReady` condition aggregates the `Ready` conditions from all Machines using `conditions.NewAggregateCondition()`. The aggregation logic:
   - If any Machine is `Ready=False`, the result is False
   - If any Machine is `Ready=Unknown` (and none are False), the result is Unknown
   - Only if all Machines are `Ready=True` is the result True

2. **Custom Merge Strategy**: The aggregation uses a custom merge strategy with specific reason codes:
   - `NotReady` when any machine is not ready
   - `Unknown` when readiness cannot be determined
   - `Ready` when all machines are ready

3. **Message Aggregation**: When not all machines are ready, the message aggregates information from the individual Machine conditions to help identify which machines have issues.

4. **No Replicas Case**: When there are no machines (e.g., scaled to 0), the condition is True with reason `NoReplicas`.

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineDeployment, MachineSets, and Machines.

6. **Related Conditions**: 
   - `MachinesUpToDate` indicates whether machines match the desired spec
   - `Available` depends on machines being ready
   - `ScalingUp`/`ScalingDown` indicate scaling activity
   - Machine's `Ready` condition is the source for aggregation
