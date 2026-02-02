# Topology Cluster Reconciler - TopologyReconciled Condition

**Reconciler**: `internal/controllers/topology/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster` (with managed topology)  
**Condition Type Modeled**: `TopologyReconciled`

## Overview

The `TopologyReconciled` condition on a Cluster indicates whether the cluster's topology (infrastructure, control plane, workers) has been successfully reconciled to match the desired state defined in the Cluster's `spec.topology` and the referenced ClusterClass. This condition is only set on Clusters that use managed topology (`spec.topology` is defined).

---

## Condition Transition Table — `TopologyReconciled` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `TopologyReconciled`) | Next reconcile (Return) |
|---|---|---|---|
| `spec.paused == true || cluster.x-k8s.io/paused annotation present` | No-op; cluster is paused | `TopologyReconciled=False; Reason=ReconcilePaused; Message="Cluster spec.paused is set to true" or "Cluster has the cluster.x-k8s.io/paused annotation"; ObservedGeneration=metadata.generation` | `none` |
| `reconcileErr != nil` | Aggregate error from reconcile phases | `TopologyReconciled=False; Reason=ReconcileFailed; Message="<error message>"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero()` | Delete owned topology resources | `TopologyReconciled=False; Reason=Deleting; Message="Cluster is deleting<. hook message if any>"; ObservedGeneration=metadata.generation` | `requeue` or `requeueAfter` (based on hooks) |
| `ClusterClass.metadata.generation != ClusterClass.status.observedGeneration` | No-op; wait for ClusterClass to be reconciled | `TopologyReconciled=False; Reason=ClusterClassNotReconciled; Message="ClusterClass not reconciled. If this condition persists please check ClusterClass status..."; ObservedGeneration=metadata.generation` | `none` |
| `!cluster.spec.infrastructureRef.IsDefined() && !cluster.spec.controlPlaneRef.IsDefined() && HookResponseTracker.AggregateRetryAfter() != 0` | No-op; BeforeClusterCreate hook is blocking | `TopologyReconciled=False; Reason=ClusterCreating; Message="<hook message>"; ObservedGeneration=metadata.generation` | `requeueAfter: <hook retryAfter>` |
| `!cluster.spec.infrastructureRef.IsDefined() && !cluster.spec.controlPlaneRef.IsDefined() && HookResponseTracker.AggregateRetryAfter() == 0` | Create infrastructure and control plane refs | `TopologyReconciled=True; Reason=ReconcileSucceeded; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.ControlPlane.IsStartingUpgrade || UpgradeTracker.ControlPlane.IsUpgrading` | Upgrade control plane; defer workers | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * <ControlPlane> upgrading to version <version>..."; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.ControlPlane.IsPendingUpgrade` | Control plane pending upgrade | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * <ControlPlane> pending upgrade to version <version>"; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachineDeployments.IsAnyUpgrading()` | Upgrade MachineDeployments | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * MachineDeployments <names> upgrading..."; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachineDeployments.IsAnyPendingUpgrade()` | MachineDeployments pending upgrade | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * MachineDeployments <names> pending upgrade..."; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachineDeployments.IsAnyUpgradeDeferred() && !hooks blocking && !CP upgrading && no MD upgrading/pending` | MachineDeployment upgrade deferred by user | `TopologyReconciled=False; Reason=MachineDeploymentsUpgradeDeferred; Message="Cluster is upgrading to <version>\n  * MachineDeployments <names> upgrade deferred using defer-upgrade annotation"; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachineDeployments.IsAnyPendingCreate()` | MachineDeployment creation deferred during CP upgrade | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * MachineDeployments <names> creation deferred while control plane upgrade is in progress"; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachinePools.IsAnyUpgrading()` | Upgrade MachinePools | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * MachinePools <names> upgrading..."; ObservedGeneration=metadata.generation` | `none` |
| `UpgradeTracker.MachinePools.IsAnyUpgradeDeferred() && conditions met` | MachinePool upgrade deferred by user | `TopologyReconciled=False; Reason=MachinePoolsUpgradeDeferred; Message="Cluster is upgrading to <version>\n  * MachinePools <names> upgrade deferred..."; ObservedGeneration=metadata.generation` | `none` |
| `hooks.IsPending(AfterClusterUpgrade, cluster)` | AfterClusterUpgrade hook pending | `TopologyReconciled=False; Reason=ClusterUpgrading; Message="Cluster is upgrading to <version>\n  * <hook message>"; ObservedGeneration=metadata.generation` | `requeueAfter: <hook retryAfter>` |
| `all resources reconciled && no upgrades in progress && no errors` | No-op; topology fully reconciled | `TopologyReconciled=True; Reason=ReconcileSucceeded; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Managed Topology Only**: This condition is only set on Clusters that have `spec.topology` defined. Clusters without managed topology do not have this condition.

2. **Upgrade Orchestration**: The topology controller orchestrates upgrades in a specific order:
   - Control plane upgrades first
   - MachineDeployments/MachinePools upgrade after control plane completes
   - Lifecycle hooks can pause upgrades at defined points

3. **Deferred Upgrades**: Users can defer upgrades for specific MachineDeployments/MachinePools using annotations:
   - `topology.cluster.x-k8s.io/defer-upgrade`
   - `topology.cluster.x-k8s.io/hold-upgrade-sequence`

4. **RuntimeSDK Hooks**: If RuntimeSDK is enabled, the following hooks can block reconciliation:
   - `BeforeClusterCreate`
   - `AfterControlPlaneInitialized`
   - `BeforeClusterUpgrade`
   - `AfterControlPlaneUpgrade`
   - `AfterClusterUpgrade`
   - `BeforeClusterDelete`

5. **ClusterClass Dependency**: The topology controller requires the referenced ClusterClass to be fully reconciled (observedGeneration == generation) before proceeding.

6. **Watch-Driven with Hook Requeue**: The reconciler is primarily watch-driven but uses `requeueAfter` when lifecycle hooks specify a retry interval.
