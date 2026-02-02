# MachinePool Reconciler - ReplicasReady Condition

**Reconciler**: `internal/controllers/machinepool/machinepool_controller.go`  
**Primary Reconciled Object**: `MachinePool`  
**Condition Type Modeled**: `ReplicasReady` (v1beta1 condition)

## Overview

The `ReplicasReady` condition on a MachinePool indicates whether all the expected replicas are ready. This condition is primarily managed by the `reconcileNodeRefs` phase which tracks the correspondence between ProviderIDs and Kubernetes Nodes.

---

## Condition Transition Table — `ReplicasReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ReplicasReady`) | Next reconcile (Return) |
|---|---|---|---|
| `status.replicas != status.readyReplicas OR len(status.nodeRefs) != status.readyReplicas` | Update status.nodeRefs from nodeRefMap; patch nodes with cluster annotations; update replica counts | `ReplicasReady=False; Reason=WaitingForReplicasReady; Message=""; ObservedGeneration=metadata.generation` | `requeueAfter: 30s` |
| `status.replicas == status.readyReplicas && len(status.nodeRefs) == status.readyReplicas && nodeRefs UIDs valid` | No-op; all replicas are ready and node references are valid | `ReplicasReady=True; Reason=ReplicasReady; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Dependency Chain**: The `ReplicasReady` condition depends on:
   - Bootstrap config being ready (`BootstrapReady` condition)
   - Infrastructure being provisioned (`InfrastructureReady` condition)
   - Nodes existing in the workload cluster with matching ProviderIDs

2. **Ready Condition (Summary)**: The MachinePool's `Ready` condition is a summary of:
   - `BootstrapReadyV1Beta1Condition`
   - `InfrastructureReadyV1Beta1Condition`
   - `ReplicasReadyV1Beta1Condition`

3. **Node Tracking**: The reconciler tracks nodes by ProviderID:
   - Builds a map of all nodes in the workload cluster
   - Matches nodes to ProviderIDs from `spec.providerIDList`
   - Deletes "retired" nodes that no longer have a corresponding ProviderID

4. **MinReadySeconds**: Node availability is computed using `spec.template.spec.minReadySeconds` when determining available replicas.

5. **Watch-Driven**: The reconciler watches:
   - MachinePool objects
   - Cluster objects (for pause state changes)
   - Nodes in the workload cluster (via ClusterCache)
   - InfraMachinePool objects
   - Bootstrap config objects

6. **RequeueAfter**: Only used when waiting for replicas to become ready (`30s` interval).

7. **Phase Transitions**: The MachinePool phase is updated based on replica counts:
   - `Pending`: Initial state
   - `Provisioning`: Bootstrap ready, infrastructure not ready
   - `Provisioned`: NodeRefs populated
   - `Running`: ReadyReplicas == desired replicas
   - `ScalingUp`/`ScalingDown`/`Scaling`: Replica mismatch
   - `Deleting`: DeletionTimestamp set

8. **MachinePool Machines**: When the InfraMachinePool controller supports it (reports `status.infrastructureMachineKind`), the reconciler creates/updates Machine objects for each infra machine, enabling individual machine tracking and health checks.
