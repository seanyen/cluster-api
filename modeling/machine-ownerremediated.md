# Machine Remediation - OwnerRemediated Condition (on Machines)

**Reconciler**: Multiple reconcilers (MachineSet, KubeadmControlPlane)  
**Primary Reconciled Object**: `Machine` (set by owning controller)  
**Condition Type Modeled**: `OwnerRemediated` (on Machine)

## Overview

The `OwnerRemediated` condition is set on Machine objects by the MachineHealthCheck controller when health check fails, and then updated by the Machine's owner (MachineSet or KubeadmControlPlane) as remediation progresses. This condition coordinates between MHC and the owner controller during the remediation lifecycle.

---

## Condition Transition Table — `OwnerRemediated` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `OwnerRemediated`) | Next reconcile (Return) |
|---|---|---|---|
| `HealthCheckSucceeded=False && OwnerRemediated not set` (MHC sets initial state) | MHC marks machine for remediation | `OwnerRemediated=False; Reason=WaitingForRemediation; Message="Waiting for remediation"; ObservedGeneration=machine.metadata.generation` | `none` |
| `OwnerRemediated=False && owner cannot remediate (e.g., maxUnhealthy exceeded)` | Owner surfaces blocking reason | `OwnerRemediated=False; Reason=CannotBeRemediated; Message="<blocking reason>"; ObservedGeneration=machine.metadata.generation` | `none` |
| `OwnerRemediated=False && owner must defer remediation` | Owner surfaces deferral reason | `OwnerRemediated=False; Reason=RemediationDeferred; Message="<deferral reason>"; ObservedGeneration=machine.metadata.generation` | `none` |
| `OwnerRemediated=False && owner starts remediation (deletes machine)` | Owner deletes machine | `OwnerRemediated=False; Reason=MachineDeleting; Message="Machine deletion in progress"; ObservedGeneration=machine.metadata.generation` | `none` |

---

## Notes

1. **Multi-Controller Coordination**: This condition is involved in a handoff:
   - **MachineHealthCheck**: Sets `HealthCheckSucceeded=False` and initializes `OwnerRemediated=False`
   - **Owner (MachineSet/KCP)**: Updates `OwnerRemediated` with remediation progress

2. **MachineSet Remediation**: For MachineSet-owned machines:
   - Checks `spec.remediation.maxUnhealthy` threshold
   - If threshold allows, deletes the unhealthy machine
   - MachineSet then creates a replacement

3. **KubeadmControlPlane Remediation**: For KCP-owned machines:
   - Respects etcd quorum requirements
   - May defer remediation to maintain cluster stability
   - Has its own remediation retry logic

4. **Blocking Reasons**: Remediation may be blocked by:
   - `maxUnhealthy` threshold exceeded
   - Etcd quorum at risk (for control plane)
   - Another machine already being remediated
   - Cluster paused

5. **Remediation Flow**:
   1. MHC detects unhealthy machine → sets `HealthCheckSucceeded=False`, `OwnerRemediated=False`
   2. Owner observes `OwnerRemediated=False` → evaluates if remediation is allowed
   3. Owner deletes machine (if allowed) → updates `OwnerRemediated` reason to `MachineDeleting`
   4. Machine deletion completes → owner creates replacement machine

6. **Watch-Driven**: The condition is updated through the normal watch-driven reconcile of MHC and the owner controller.
