# KubeadmControlPlane Reconciler - EtcdClusterHealthy Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `EtcdClusterHealthy`

## Overview

The `EtcdClusterHealthy` condition on a KubeadmControlPlane indicates whether the etcd cluster is healthy. It aggregates the health status of individual etcd members from all control plane Machines. This condition is only set when etcd is managed by KCP (not external etcd).

---

## Condition Transition Table — `EtcdClusterHealthy` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `EtcdClusterHealthy`) | Next reconcile (Return) |
|---|---|---|---|
| `etcd is external (not managed by KCP)` | No-op; external etcd tools manage this condition | (condition not set by KCP) | `none` |
| `failed to list control plane Nodes` | Surface inspection failure | `EtcdClusterHealthy=Unknown; Reason=InspectionFailed; Message="Failed to get Nodes hosting the etcd cluster"; ObservedGeneration=metadata.generation` | `none` |
| `failed to connect to any etcd member` | Surface connection failure on all machines | `EtcdClusterHealthy=Unknown; Reason=HealthUnknown; Message="Failed to connect to etcd: <error>"; ObservedGeneration=metadata.generation` | `none` |
| `all etcd members healthy && members match machines` | Aggregate conditions from machines | `EtcdClusterHealthy=True; Reason=EtcdClusterHealthy; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any etcd member unhealthy or has alarms` | Aggregate conditions; surface issues | `EtcdClusterHealthy=False; Reason=EtcdClusterNotHealthy; Message="<aggregated issues from etcd members>"; ObservedGeneration=metadata.generation` | `none` |
| `any etcd member status unknown` | Aggregate conditions; surface unknown status | `EtcdClusterHealthy=Unknown; Reason=HealthUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |
| `etcd members don't match machines` | Surface mismatch | `EtcdClusterHealthy=False; Reason=EtcdClusterNotHealthy; Message="Etcd members do not match Machines: <details>"; ObservedGeneration=metadata.generation` | `none` |
| `control plane Node without corresponding Machine` | Surface orphaned node | `EtcdClusterHealthy=False; Reason=EtcdClusterNotHealthy; Message="Control plane Node <name> does not have a corresponding Machine"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Managed vs External etcd**: This condition is only actively managed when etcd is managed by KCP. For external etcd, KCP does not set this condition, allowing external tools to manage it.

2. **Aggregation Source**: The condition aggregates `EtcdMemberHealthy` conditions from all control plane `Machine` objects.

3. **Etcd Member Health Checks**: Each etcd member is checked for:
   - Membership in the cluster
   - Absence of alarms (NOSPACE, CORRUPT, etc.)
   - Connectivity and responsiveness

4. **Member-Machine Matching**: The condition also verifies that:
   - Each Machine has a corresponding etcd member
   - Each etcd member has a corresponding Machine
   - No orphaned Nodes exist without a Machine

5. **Impact on Available**: The `EtcdClusterHealthy` condition affects the `Available` condition - if etcd is not healthy, the control plane may not be considered available (depends on quorum).

6. **Watch-Driven**: The reconciler is watch-driven via the ClusterCache to observe etcd member status in the workload cluster.
