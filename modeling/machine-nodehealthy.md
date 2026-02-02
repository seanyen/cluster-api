# Machine Reconciler - NodeHealthy Condition

**Reconciler**: `internal/controllers/machine/machine_controller.go`  
**Primary Reconciled Object**: `Machine`  
**Condition Type Modeled**: `NodeHealthy`

## Overview

The `NodeHealthy` condition on a Machine indicates whether the corresponding Kubernetes Node is healthy. This condition summarizes all node conditions (not just `Ready`) and accounts for connection issues to the workload cluster.

---

## Condition Transition Table — `NodeHealthy` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `NodeHealthy`) | Next reconcile (Return) |
|---|---|---|---|
| `cluster.status.initialization.infrastructureProvisioned == false` | No-op; waiting for cluster infrastructure | `NodeHealthy=Unknown; Reason=InspectionFailed; Message="Waiting for Cluster status.initialization.infrastructureProvisioned to be true"; ObservedGeneration=metadata.generation` | `none` |
| `ClusterControlPlaneInitializedCondition != True` | No-op; waiting for control plane | `NodeHealthy=Unknown; Reason=InspectionFailed; Message="Waiting for Cluster control plane to be initialized"; ObservedGeneration=metadata.generation` | `none` |
| `healthCheckingState.LastProbeSuccessTime.IsZero() && consecutiveFailures < 5 && condition not set` | No-op; connection not established yet | `NodeHealthy=Unknown; Reason=ConnectionDown; Message="Remote connection not established yet"; ObservedGeneration=metadata.generation` | `none` |
| `time since lastProbeSuccess > remoteConditionsGracePeriod` | Overwrite to connection down | `NodeHealthy=Unknown; Reason=ConnectionDown; Message="Last successful probe at <time>"; ObservedGeneration=metadata.generation` | `none` |
| `nodeGetErr == ErrClusterNotConnected && condition not set` | No-op; temporary connection issue | `NodeHealthy=Unknown; Reason=ConnectionDown; Message="Last successful probe at <time>"; ObservedGeneration=metadata.generation` | `none` |
| `nodeGetErr != nil && nodeGetErr != ErrClusterNotConnected` | No-op; internal error | `NodeHealthy=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `node exists && all node conditions healthy` | Summarize node conditions | `NodeHealthy=True; Reason=Healthy; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `node exists && some node conditions unhealthy` | Summarize node conditions | `NodeHealthy=False; Reason=Unhealthy; Message="* <unhealthy condition details>"; ObservedGeneration=metadata.generation` | `none` |
| `node exists && some node conditions unknown` | Summarize node conditions | `NodeHealthy=Unknown; Reason=Unknown; Message="* <unknown condition details>"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp set && nodeRef defined` | No-op; node deleted during machine deletion | `NodeHealthy=False; Reason=Deleted; Message="Node <name> has been deleted"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp set && nodeRef not defined` | No-op; node never existed | `NodeHealthy=Unknown; Reason=DoesNotExist; Message="Node does not exist"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && nodeRef defined` | No-op; node unexpectedly deleted | `NodeHealthy=False; Reason=Deleted; Message="Node <name> has been deleted while the Machine still exists"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && providerID set` | No-op; waiting for node to appear | `NodeHealthy=Unknown; Reason=InspectionFailed; Message="Waiting for a Node with spec.providerID <id> to exist"; ObservedGeneration=metadata.generation` | `none` |
| `node not found && DeletionTimestamp not set && providerID not set` | No-op; waiting for provider ID | `NodeHealthy=Unknown; Reason=InspectionFailed; Message="Waiting for <Kind> to report spec.providerID"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Node Condition Summary**: The `NodeHealthy` condition summarizes all Kubernetes Node conditions (MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable, etc.), not just the `Ready` condition.

2. **Connection Grace Period**: There's a grace period (`remoteConditionsGracePeriod`) during which the controller tolerates connection issues before overwriting conditions to `ConnectionDown`.

3. **Last Known Status**: When experiencing temporary connection issues, the controller keeps the last known status rather than immediately flipping to Unknown.

4. **Related NodeReady Condition**: The `NodeReady` condition specifically mirrors the Node's `Ready` condition, while `NodeHealthy` is a broader summary.

5. **Cluster Cache**: The controller uses the cluster cache to connect to workload clusters and retrieve Node information.

6. **Watch-Driven**: The reconciler watches the Machine and uses the cluster cache for Node changes.

7. **Related Conditions**: 
   - `NodeReady` mirrors the specific Node Ready condition
   - `BootstrapConfigReady` and `InfrastructureReady` indicate provisioning status
   - `Ready` is a summary that includes node health
