# KubeadmControlPlane Reconciler - ControlPlaneComponentsHealthy Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `ControlPlaneComponentsHealthy`

## Overview

The `ControlPlaneComponentsHealthy` condition on a KubeadmControlPlane indicates whether the Kubernetes control plane components (API server, controller-manager, scheduler) are healthy across all control plane machines. It aggregates the health status of individual component pods from all control plane Machines.

---

## Condition Transition Table — `ControlPlaneComponentsHealthy` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneComponentsHealthy`) | Next reconcile (Return) |
|---|---|---|---|
| `failed to list control plane Nodes` | Surface inspection failure | `ControlPlaneComponentsHealthy=Unknown; Reason=InspectionFailed; Message="Failed to get Nodes hosting control plane components"; ObservedGeneration=metadata.generation` | `none` |
| `failed to get control plane Pods` | Surface inspection failure | `ControlPlaneComponentsHealthy=Unknown; Reason=InspectionFailed; Message="Failed to get control plane Pods"; ObservedGeneration=metadata.generation` | `none` |
| `all control plane components healthy on all machines` | Aggregate conditions from machines | `ControlPlaneComponentsHealthy=True; Reason=ControlPlaneComponentsHealthy; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any control plane component unhealthy` | Aggregate conditions; surface issues | `ControlPlaneComponentsHealthy=False; Reason=ControlPlaneComponentsNotHealthy; Message="<aggregated issues from components>"; ObservedGeneration=metadata.generation` | `none` |
| `any control plane component status unknown` | Aggregate conditions; surface unknown status | `ControlPlaneComponentsHealthy=Unknown; Reason=HealthUnknown; Message="<aggregated unknown reasons>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Control Plane Components Checked**: The condition checks health of:
   - `kube-apiserver` Pod on each control plane node
   - `kube-controller-manager` Pod on each control plane node
   - `kube-scheduler` Pod on each control plane node
   - `etcd` Pod on each control plane node (if etcd is managed)

2. **Pod Health Criteria**: A control plane component Pod is considered healthy when:
   - The Pod exists
   - The Pod is in `Running` phase
   - All containers in the Pod are ready

3. **Machine-Level Conditions**: Individual machine conditions are set for each component:
   - `APIServerPodHealthy`
   - `ControllerManagerPodHealthy`
   - `SchedulerPodHealthy`
   - `EtcdPodHealthy` (if etcd is managed)

4. **Aggregation Source**: The KCP-level condition aggregates the component health conditions from all control plane Machines.

5. **Impact on Available**: The `ControlPlaneComponentsHealthy` condition affects the `Available` condition - at least one healthy control plane is required for availability.

6. **Watch-Driven**: The reconciler is watch-driven via the ClusterCache to observe Pod status in the workload cluster.

7. **kubeadm Layout Consideration**: The condition considers kubeadm's control plane layout:
   - API server connects only to local etcd member
   - Controller-manager and scheduler connect to local API server
   - Therefore, a machine is considered healthy only when ALL its components are healthy
