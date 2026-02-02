# Machine Reconciler - Available Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `Available`

## Overview

The `Available` condition on a Machine indicates whether the Machine is ready and has been ready for at least `spec.minReadySeconds`. This condition is derived from the `Ready` condition and a time-based check.

---

## Condition Transition Table — `Available` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Available`) | Next reconcile (Return) |
|---|---|---|---|
| `Ready condition is nil` | Set Available to Unknown (internal error, should not happen) | `Available=Unknown; Reason=AvailableInternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `Ready=False` | No-op; Machine not ready so not available | `Available=False; Reason=NotReady; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `Ready=Unknown` | No-op; Machine readiness unknown so availability unknown | `Available=False; Reason=NotReady; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `Ready=True && time.Since(Ready.LastTransitionTime) < spec.minReadySeconds` | No-op; Machine is ready but minReadySeconds not yet elapsed | `Available=False; Reason=WaitingForMinReadySeconds; Message=""; ObservedGeneration=metadata.generation` | `requeueAfter: remaining time until minReadySeconds` |
| `Ready=True && time.Since(Ready.LastTransitionTime) >= spec.minReadySeconds` | No-op; Machine is ready and minReadySeconds elapsed | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `Ready=True && spec.minReadySeconds == nil (defaults to 0)` | No-op; Machine is ready and no minReadySeconds configured | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Dependency on Ready**: The `Available` condition is directly derived from the `Ready` condition. The `Ready` condition is a summary of:
   - `Deleting` (negative polarity)
   - `Updating` (negative polarity)
   - `BootstrapConfigReady`
   - `InfrastructureReady`
   - `NodeHealthy`
   - `HealthCheckSucceeded` (tolerated if missing)
   - Any custom readiness gates from `spec.readinessGates`

2. **MinReadySeconds**: If `spec.minReadySeconds` is set, the Machine must be `Ready=True` for that duration before becoming `Available=True`. This prevents flapping during brief transient ready states.

3. **Requeue Logic**: When waiting for `minReadySeconds` to elapse, the controller returns a `requeueAfter` with the remaining time.

4. **Watch-Driven**: Apart from the `minReadySeconds` case, the reconciler is watch-driven.
