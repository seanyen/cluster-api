# Machine Reconciler - NodeReady Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `NodeReady`

## Overview

The `NodeReady` condition on a Machine mirrors the `Ready` condition from the corresponding Kubernetes Node. This condition specifically tracks whether the Node's `Ready` condition is True, while accounting for connection issues to the workload cluster.

---

## Condition Transition Table — `NodeReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `NodeReady`) | Next reconcile (Return) |
|---|---|---|---|
| `cluster.status.initialization.infrastructureProvisioned == false` | No-op; waiting for cluster infrastructure | `NodeReady=Unknown; Reason=InspectionFailed; Message="Waiting for Cluster status.initialization.infrastructureProvisioned to be true"; ObservedGeneration=metadata.generation` | `none` |
| `ClusterControlPlaneInitializedCondition != True` | No-op; waiting for control plane | `NodeReady=Unknown; Reason=InspectionFailed; Message="Waiting for Cluster control plane to be initialized"; ObservedGeneration=metadata.generation` | `none` |
| `healthCheckingState.LastProbeSuccessTime.IsZero() && consecutiveFailures < 5 && condition not set` | No-op; connection not established yet | `NodeReady=Unknown; Reason=ConnectionDown; Message="Remote connection not established yet"; ObservedGeneration=metadata.generation` | `none` |
| `time since lastProbeSuccess > remoteConditionsGracePeriod` | Overwrite to connection down | `NodeReady=Unknown; Reason=ConnectionDown; Message="Last successful probe at <time>"; ObservedGeneration=metadata.generation` | `none` |
| `nodeGetErr == ErrClusterNotConnected && condition not set` | No-op; temporary connection issue | `NodeReady=Unknown; Reason=ConnectionDown; Message="Last successful probe at <time>"; ObservedGeneration=metadata.generation` | `none` |
| `nodeGetErr != nil && nodeGetErr != ErrClusterNotConnected` | No-op; internal error | `NodeReady=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `node exists && Node.Ready == True` | Mirror Node Ready condition | `NodeReady=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `node exists && Node.Ready == False` | Mirror Node Ready condition | `NodeReady=False; Reason=NotReady; Message="* Node.Ready: <node condition message>"; ObservedGeneration=metadata.generation` | `none` |
| `node exists && Node.Ready == Unknown` | Mirror Node Ready condition | `NodeReady=Unknown; Reason=Unknown; Message="* Node.Ready: <node condition message>"; ObservedGeneration=metadata.generation` | `none` |
| `node exists && Node.Ready condition not present` | No-op; condition not reported yet | `NodeReady=Unknown; Reason=Unknown; Message="* Node.Ready: Condition not yet reported"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp set && nodeRef defined` | No-op; node deleted during machine deletion | `NodeReady=False; Reason=Deleted; Message="Node <name> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp set && nodeRef not defined` | No-op; node never existed | `NodeReady=Unknown; Reason=DoesNotExist; Message="Node does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && nodeRef defined` | No-op; node unexpectedly deleted | `NodeReady=False; Reason=Deleted; Message="Node <name> has been deleted while the Machine still exists"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && providerID set` | No-op; waiting for node to appear | `NodeReady=Unknown; Reason=InspectionFailed; Message="Waiting for a Node with spec.providerID <id> to exist"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && providerID not set` | No-op; waiting for provider ID | `NodeReady=Unknown; Reason=InspectionFailed; Message="Waiting for <Kind> to report spec.providerID"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Node Ready Mirror**: This condition directly mirrors the Kubernetes Node's `Ready` condition status, while the `NodeHealthy` condition summarizes all node conditions.

2. **Connection Grace Period**: There's a grace period (`remoteConditionsGracePeriod`) during which the controller tolerates connection issues before overwriting conditions to `ConnectionDown`.

3. **Last Known Status**: When experiencing temporary connection issues, the controller keeps the last known status rather than immediately flipping to Unknown.

4. **Condition Set Together**: The `NodeReady` and `NodeHealthy` conditions are set together by `setNodeHealthyAndReadyConditions()` but track different aspects of node health.

5. **Cluster Cache**: The controller uses the cluster cache to connect to workload clusters and retrieve Node information.

6. **Watch-Driven**: The reconciler watches the Machine and uses the cluster cache for Node changes.

7. **Related Conditions**: 
   - `NodeHealthy` summarizes all Node conditions
   - `Ready` is a summary that includes node readiness
