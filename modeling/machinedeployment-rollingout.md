# MachineDeployment Reconciler - RollingOut Condition

**Reconciler**: `internal/controllers/machinedeployment/machinedeployment_controller.go`  
**Primary Reconciled Object**: `MachineDeployment`  
**Condition Type Modeled**: `RollingOut`

## Overview

The `RollingOut` condition on a MachineDeployment indicates whether the MachineDeployment is actively rolling out changes to its Machines. A rollout occurs when there are Machines that are not up-to-date with the current MachineDeployment spec. This condition helps operators understand if a rollout is in progress and the reasons for the rollout.

---

## Condition Transition Table — `RollingOut` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RollingOut`) | Next reconcile (Return) |
|---|---|---|---|
| `rollingOutReplicas == 0 (no machines with UpToDate=False)` | No-op; all machines are up-to-date | `RollingOut=False; Reason=NotRollingOut; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `rollingOutReplicas > 0 && rolloutReasons.Len() == 0` | Continue rollout; create/update MachineSets as needed | `RollingOut=True; Reason=RollingOut; Message="Rolling out <n> not up-to-date replicas"; ObservedGeneration=metadata.generation` | `none` |
| `rollingOutReplicas > 0 && rolloutReasons includes version change` | Continue rollout; version change prioritized in message | `RollingOut=True; Reason=RollingOut; Message="Rolling out <n> not up-to-date replicas\n* Version: <old> → <new>\n* <other reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `rollingOutReplicas > 0 && rolloutReasons includes multiple reasons (no version change)` | Continue rollout; reasons sorted alphabetically | `RollingOut=True; Reason=RollingOut; Message="Rolling out <n> not up-to-date replicas\n* <reason1>\n* <reason2>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Rolling Out Replicas Count**: `rollingOutReplicas` is calculated by counting Machines where `UpToDate` condition is `False`. Machines without an `UpToDate` condition or with `UpToDate=True` are not counted.

2. **Rollout Reasons**: The condition message includes the reasons why machines are not up-to-date, extracted from the `UpToDate` condition's message on each machine. Common reasons include:
   - Version change (Kubernetes version upgrade)
   - Template changes (infrastructure or bootstrap template changes)
   - Configuration changes

3. **Version Change Priority**: If a version change is among the rollout reasons, it is sorted to appear first in the message, as version changes are typically the most significant type of update.

4. **Message Aggregation**: The controller collects unique rollout reasons from all not-up-to-date machines. Under normal circumstances, all machines are rolling out for the same reasons, but during rapid changes, different machines may have different reasons.

5. **Watch-Driven**: The reconciler is watch-driven based on changes to the MachineDeployment, its owned MachineSets, and the Machines.

6. **Related Conditions**:
   - `ScalingUp` indicates whether the MachineDeployment is scaling up
   - `ScalingDown` indicates whether the MachineDeployment is scaling down
   - `MachinesReady` indicates whether owned Machines are ready
   - `MachinesUpToDate` indicates whether owned Machines are up-to-date
   - `Available` indicates overall availability of the MachineDeployment
