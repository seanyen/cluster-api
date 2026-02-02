# MachineHealthCheck Reconciler - HealthCheckSucceeded Condition (on Machines)

**Reconciler**: `internal/controllers/machinehealthcheck/machinehealthcheck_controller.go`  
**Primary Reconciled Object**: `MachineHealthCheck` (sets condition on `Machine`)  
**Condition Type Modeled**: `HealthCheckSucceeded` (on Machine)

## Overview

The `HealthCheckSucceeded` condition is set by the MachineHealthCheck controller on Machine objects to indicate whether the Machine passed its health checks. This condition is used to track machine health status and trigger remediation when needed.

---

## Condition Transition Table — `HealthCheckSucceeded` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `HealthCheckSucceeded`) | Next reconcile (Return) |
|---|---|---|---|
| `machine has cluster.x-k8s.io/remediate-machine annotation` | Mark machine for remediation | `HealthCheckSucceeded=False; Reason=HasRemediateMachineAnnotation; Message="Marked for remediation via remediate-machine annotation"; ObservedGeneration=machine.metadata.generation` | `none` |
| `machine.status.nodeRef not set && nodeStartupTimeout not exceeded` | No-op; waiting for node | `HealthCheckSucceeded=Unknown; Reason=WaitingForNodeRef; Message="Waiting for Node to be created"; ObservedGeneration=machine.metadata.generation` | `requeueAfter: nodeStartupTimeout - elapsed` |
| `machine.status.nodeRef not set && nodeStartupTimeout exceeded` | Mark machine unhealthy | `HealthCheckSucceeded=False; Reason=NodeStartupTimedOut; Message="Node failed to start within <timeout>"; ObservedGeneration=machine.metadata.generation` | `none` |
| `node exists && all unhealthyConditions pass (healthy)` | Mark machine healthy | `HealthCheckSucceeded=True; Reason=Succeeded; Message=""; ObservedGeneration=machine.metadata.generation` | `none` |
| `node exists && any unhealthyCondition fails && timeout not exceeded` | No-op; in grace period | `HealthCheckSucceeded=Unknown; Reason=NodeConditionsNotYetUnhealthy; Message="Waiting for unhealthyCondition timeout"; ObservedGeneration=machine.metadata.generation` | `requeueAfter: condition timeout - elapsed` |
| `node exists && any unhealthyCondition fails && timeout exceeded` | Mark machine unhealthy | `HealthCheckSucceeded=False; Reason=<ConditionType>Unhealthy; Message="Node condition <type> is <status> for more than <timeout>"; ObservedGeneration=machine.metadata.generation` | `none` |
| `node not found && machine not deleting` | Mark machine unhealthy | `HealthCheckSucceeded=False; Reason=NodeNotFound; Message="Node not found"; ObservedGeneration=machine.metadata.generation` | `none` |

---

## Notes

1. **Condition Location**: This condition is set on Machine objects, not on the MachineHealthCheck object itself. The MHC object has its own `RemediationAllowed` condition.

2. **Unhealthy Conditions**: The MachineHealthCheck evaluates node conditions specified in `spec.unhealthyConditions`, each with:
   - `type`: The Node condition type to check
   - `status`: The status value that indicates unhealthy
   - `timeout`: How long the condition must persist before marking unhealthy

3. **Node Startup Timeout**: Configured via `spec.nodeStartupTimeout`, this is how long to wait for a Machine's Node to appear before marking it unhealthy.

4. **Remediate Machine Annotation**: Machines with the `cluster.x-k8s.io/remediate-machine` annotation are immediately marked for remediation.

5. **Impact on Remediation**: When `HealthCheckSucceeded=False`, the MHC controller sets `OwnerRemediated=False` on the Machine, signaling the Machine's owner to remediate it.

6. **Watch-Driven with Timeouts**: The reconciler is watch-driven but uses `requeueAfter` to recheck conditions after timeout periods.
