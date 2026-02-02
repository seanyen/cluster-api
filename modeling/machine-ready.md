# Machine Reconciler - Ready Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `Ready`

## Overview

The `Ready` condition on a Machine is a **summary condition** that aggregates the status of multiple sub-conditions. It indicates whether the Machine has successfully completed its bootstrap process, infrastructure provisioning, and the associated Node is healthy.

The summary considers the following conditions (with polarity):
- `Deleting` (negative polarity) - whether Machine is being deleted
- `Updating` (negative polarity) - whether Machine is being updated
- `BootstrapConfigReady` (positive polarity) - whether bootstrap config is ready
- `InfrastructureReady` (positive polarity) - whether infrastructure is ready
- `NodeHealthy` (positive polarity) - whether the Node is healthy
- `HealthCheckSucceeded` (positive polarity, tolerated if missing) - whether health checks pass
- Custom readiness gates from `spec.readinessGates`

---

## Condition Transition Table — `Ready` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Ready`) | Next reconcile (Return) |
|---|---|---|---|
| `Error computing summary condition` | Log error; set Unknown condition | `Ready=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `Deleting=True (machine.DeletionTimestamp != nil)` | Compute summary with Deleting condition override for cleaner aggregation | `Ready=False; Reason=Deleting; Message="Machine deletion in progress, stage: <DeletingReason>"; ObservedGeneration=metadata.generation` | `none` |
| `Deleting=True && deletion > 15min && stage=DrainingNode && drain > 5min with PDB violations` | Compute summary with detailed delay message | `Ready=False; Reason=Deleting; Message="Machine deletion in progress since more than 15m, stage: DrainingNode, delay likely due to PodDisruptionBudgets"; ObservedGeneration=metadata.generation` | `none` |
| `Updating=True` | Compute summary; Updating blocks ready | `Ready=False; Reason=Updating; Message=<updating details>; ObservedGeneration=metadata.generation` | `none` |
| `BootstrapConfigReady=False OR BootstrapConfigReady=Unknown` | Compute summary; bootstrap not ready blocks ready | `Ready=False; Reason=BootstrapConfigNotReady; Message=<bootstrap condition message>; ObservedGeneration=metadata.generation` | `none` |
| `InfrastructureReady=False OR InfrastructureReady=Unknown` | Compute summary; infra not ready blocks ready | `Ready=False; Reason=InfrastructureNotReady; Message=<infra condition message>; ObservedGeneration=metadata.generation` | `none` |
| `NodeHealthy=False OR NodeHealthy=Unknown` | Compute summary; unhealthy node blocks ready | `Ready=False; Reason=NodeNotHealthy; Message=<node condition message>; ObservedGeneration=metadata.generation` | `none` |
| `HealthCheckSucceeded=False (if present)` | Compute summary; failed health check blocks ready | `Ready=False; Reason=HealthCheckFailed; Message=<health check message>; ObservedGeneration=metadata.generation` | `none` |
| `Any custom readinessGate condition is False/Unknown (positive polarity) or True (negative polarity)` | Compute summary; custom gate blocks ready | `Ready=False; Reason=<gate condition reason>; Message=<gate condition message>; ObservedGeneration=metadata.generation` | `none` |
| `All conditions satisfied (Deleting=False, Updating=False, BootstrapConfigReady=True, InfrastructureReady=True, NodeHealthy=True, HealthCheckSucceeded=True/missing, all gates pass)` | Compute summary; all conditions healthy | `Ready=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Summary Condition**: `Ready` is computed using `conditions.NewSummaryCondition()` which merges the status of multiple sub-conditions.

2. **Negative Polarity Conditions**: `Deleting` and `Updating` conditions use negative polarity - when `True`, they indicate a problem (machine is being deleted/updated). Custom readiness gates can also specify negative polarity via `spec.readinessGates[].polarity`.

3. **HealthCheckSucceeded Tolerance**: The `HealthCheckSucceeded` condition is tolerated if missing (via `IgnoreTypesIfMissing`). If a MachineHealthCheck is not configured, the absence of this condition won't block `Ready=True`.

4. **Deleting Condition Override**: When the machine is being deleted, the controller calculates a custom deleting condition for the summary to:
   - Avoid verbose details in the summary message
   - Enable nice aggregation of Ready conditions from many Machines into MachinesReady condition on MachineSet/MachineDeployment
   - Surface delay reasons (PDBs, pods not terminating, etc.) only after 5+ minutes of drain time

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the Machine, BootstrapConfig, InfraMachine, and Node resources.

6. **Downstream Impact**: The `Ready` condition is used by the `Available` condition (Machine must be Ready for minReadySeconds to become Available) and is aggregated up to MachineSet and MachineDeployment `MachinesReady` conditions.
