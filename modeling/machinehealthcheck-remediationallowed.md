# MachineHealthCheck Reconciler - RemediationAllowed Condition

**Reconciler**: `internal/controllers/machinehealthcheck/machinehealthcheck_controller.go`  
**Primary Reconciled Object**: `MachineHealthCheck`  
**Condition Type Modeled**: `RemediationAllowed`

## Overview

The `RemediationAllowed` condition on a MachineHealthCheck indicates whether the MachineHealthCheck is allowed to trigger remediation for unhealthy machines. This is based on the number of unhealthy machines relative to the configured `maxUnhealthy` threshold or `unhealthyRange`.

---

## Condition Transition Table — `RemediationAllowed` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RemediationAllowed`) | Next reconcile (Return) |
|---|---|---|---|
| `unhealthyMachines > maxUnhealthy (when spec.remediation.triggerIf.unhealthyLessThanOrEqualTo is set)` | No-op; set RemediationsAllowed=0 in status; emit RemediationRestricted event if unhealthy > 0 | `RemediationAllowed=False; Reason=TooManyUnhealthy; Message="Remediation is not allowed, the number of not started or unhealthy machines exceeds maxUnhealthy (total: <N>, unhealthy: <M>, maxUnhealthy: <X>)"; ObservedGeneration=metadata.generation` | `none` |
| `unhealthyMachines not in unhealthyRange (when spec.remediation.triggerIf.unhealthyInRange is set)` | No-op; set RemediationsAllowed=0 in status; emit RemediationRestricted event if unhealthy > 0 | `RemediationAllowed=False; Reason=TooManyUnhealthy; Message="Remediation is not allowed, the number of not started or unhealthy machines does not fall within the range (total: <N>, unhealthy: <M>, unhealthyRange: <X>)"; ObservedGeneration=metadata.generation` | `none` |
| `unhealthyMachines <= maxUnhealthy (when spec.remediation.triggerIf.unhealthyLessThanOrEqualTo is set)` | Set RemediationsAllowed in status; patch unhealthy machines with OwnerRemediated=False or create external remediation request | `RemediationAllowed=True; Reason=RemediationAllowed; Message=""; ObservedGeneration=metadata.generation` | `requeueAfter: minNextCheckTime` (if targets may become unhealthy) or `none` |
| `unhealthyMachines in unhealthyRange (when spec.remediation.triggerIf.unhealthyInRange is set)` | Set RemediationsAllowed in status; patch unhealthy machines with OwnerRemediated=False or create external remediation request | `RemediationAllowed=True; Reason=RemediationAllowed; Message=""; ObservedGeneration=metadata.generation` | `requeueAfter: minNextCheckTime` (if targets may become unhealthy) or `none` |
| `totalTargets == 0 (no machines match selector)` | Set ExpectedMachines=0, CurrentHealthy=0 | `RemediationAllowed=True; Reason=RemediationAllowed; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **maxUnhealthy Calculation**: 
   - `maxUnhealthy` can be an absolute number or a percentage of total targets
   - Default is `100%` (remediation always allowed)
   - If `maxUnhealthy=0`, any unhealthy machine blocks remediation

2. **unhealthyRange Alternative**:
   - Instead of `maxUnhealthy`, users can specify `unhealthyRange` (e.g., `1-3`)
   - Remediation is only allowed if the number of unhealthy machines falls within the range

3. **Health Check Logic**: A machine is considered unhealthy if:
   - Its Node has an unhealthy condition (matching `spec.unhealthyConditions`) for longer than the specified timeout
   - Its Node doesn't exist and `nodeStartupTimeout` has elapsed since machine creation

4. **Remediation Actions**:
   - If `spec.remediation.templateRef` is set: creates an external remediation request object
   - Otherwise: sets `MachineOwnerRemediatedCondition=False` on the Machine for the owner controller to handle

5. **Status Fields Updated**:
   - `status.expectedMachines`: Total number of machines matching the selector
   - `status.currentHealthy`: Number of healthy machines
   - `status.remediationsAllowed`: Number of additional remediations allowed before hitting maxUnhealthy
   - `status.targets`: List of machine names being health-checked

6. **Watch-Driven with Requeue**: The reconciler uses watches on Machines and may requeue based on when the next health check timeout will expire.
