# KubeadmControlPlane Reconciler - ControlPlaneComponentsHealthy Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `ControlPlaneComponentsHealthy`

## Overview

The `ControlPlaneComponentsHealthy` condition on a KubeadmControlPlane indicates whether the Kubernetes control plane components (API server, controller-manager, scheduler, and optionally etcd) are healthy across all control plane machines. It is computed by aggregating the health status of individual component pods from all control plane Machines via `reconcileControlPlaneAndMachinesConditions` → `workloadCluster.UpdateStaticPodConditions`.

---

## Condition Transition Table — `ControlPlaneComponentsHealthy` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ControlPlaneComponentsHealthy`) | Next reconcile (Return) |
|---|---|---|---|
| `!status.initialization.controlPlaneInitialized OR Initialized condition != True` | Set condition to Unknown; cannot inspect workload cluster | `ControlPlaneComponentsHealthy=Unknown; Reason=InspectionFailed; Message="Waiting for Cluster control plane to be initialized"; ObservedGeneration=metadata.generation` | `none` |
| `ClusterCache.LastProbeSuccessTime.IsZero() && ConsecutiveFailures < 5` | Set condition to Unknown if not already set (preserve last known state) | `ControlPlaneComponentsHealthy=Unknown; Reason=ConnectionDown; Message="Remote connection not established yet"; ObservedGeneration=metadata.generation` | `requeue (error)` |
| `time.Since(max(LastProbeSuccessTime, Initialized.LastTransitionTime)) > RemoteConditionsGracePeriod` | Overwrite condition to Unknown; connection has been down too long | `ControlPlaneComponentsHealthy=Unknown; Reason=ConnectionDown; Message="Last successful probe at <timestamp>"; ObservedGeneration=metadata.generation` | `requeue (error)` |
| `workloadCluster connection error (ErrClusterNotConnected)` | Set condition to Unknown if not already set (preserve last known state) | `ControlPlaneComponentsHealthy=Unknown; Reason=ConnectionDown; Message="Last successful probe at <timestamp>"; ObservedGeneration=metadata.generation` | `requeue (error)` |
| `workloadCluster connection error (other error)` | Overwrite condition to Unknown | `ControlPlaneComponentsHealthy=Unknown; Reason=InspectionFailed; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `requeue (error)` |
| `failed to list control plane Nodes in workload cluster` | Set condition to Unknown on all machines and KCP | `ControlPlaneComponentsHealthy=Unknown; Reason=InspectionFailed; Message="Failed to get Nodes hosting control plane components: <error>"; ObservedGeneration=metadata.generation` | `none` |
| `node without corresponding Machine exists (and no provisioning machines)` | Surface KCP-level error | `ControlPlaneComponentsHealthy=False; Reason=NotHealthy; Message="* Control plane Node <name> does not have a corresponding Machine"; ObservedGeneration=metadata.generation` | `none` |
| `at least one Machine has False status for any component condition` | Aggregate machine conditions; surface issues | `ControlPlaneComponentsHealthy=False; Reason=NotHealthy; Message="* Machine(s) <names>:\n  * <Condition>: <message>"; ObservedGeneration=metadata.generation` | `none` |
| `at least one Machine (with ProviderID set) has Unknown status for any component condition AND no False conditions` | Aggregate machine conditions; surface unknown | `ControlPlaneComponentsHealthy=Unknown; Reason=HealthUnknown; Message="* Machine(s) <names>:\n  * <Condition>: <message>"; ObservedGeneration=metadata.generation` | `none` |
| `all Machines have True status for all component conditions` | Aggregate machine conditions; all healthy | `ControlPlaneComponentsHealthy=True; Reason=Healthy; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `no Machines exist or no Machines reporting component status` | Set condition to Unknown | `ControlPlaneComponentsHealthy=Unknown; Reason=HealthUnknown; Message="No Machines reporting control plane status"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Control Plane Components Checked**: The condition aggregates health of static pods:
   - `APIServerPodHealthy` - kube-apiserver Pod on each control plane node
   - `ControllerManagerPodHealthy` - kube-controller-manager Pod on each control plane node
   - `SchedulerPodHealthy` - kube-scheduler Pod on each control plane node
   - `EtcdPodHealthy` - etcd Pod on each control plane node (only if etcd is managed)

2. **Pod Health Criteria**: A control plane component Pod is considered healthy when:
   - The Pod exists in the `kube-system` namespace
   - The Pod is in `Running` phase
   - All containers in the Pod are ready (PodReady condition is True)

3. **Pod Status Mapping**:
   - `Running` phase + `PodReady=True` → Machine condition `True`, Reason=`PodRunning`
   - `Pending` phase → Machine condition `False`, Reason=`PodProvisioning`
   - Pod not found → Machine condition `False`, Reason=`PodDoesNotExist`
   - `Failed`/`Succeeded` phase → Machine condition `False`, Reason=`PodFailed`
   - Node unreachable taint → Machine condition `Unknown`, Reason=`PodInspectionFailed`
   - Node Ready=Unknown → Machine condition `Unknown`, Reason=`PodInspectionFailed`

4. **Aggregation Logic**: The KCP-level condition is computed from Machine conditions:
   - If any Machine has `False` → KCP condition is `False`
   - Else if any Machine (with ProviderID) has `Unknown` → KCP condition is `Unknown`
   - Else if any Machine has `True` → KCP condition is `True`
   - Machines without ProviderID are treated as `True` (still provisioning)

5. **Connection Grace Period**: The `RemoteConditionsGracePeriod` flag (default configurable) determines how long to preserve last known state during connection issues before overwriting to Unknown.

6. **Watch-Driven**: The reconciler is watch-driven via the ClusterCache to observe Pod status in the workload cluster. No explicit requeueAfter is used; reconciliation is triggered by Pod changes.

7. **Impact on Available**: The `ControlPlaneComponentsHealthy` condition affects the `Available` condition - at least one Machine with healthy control plane components is required for KCP availability.

8. **kubeadm Layout Consideration**: The condition considers kubeadm's control plane layout where all components run as static pods on each control plane node. A machine is considered healthy only when ALL its components are healthy.
