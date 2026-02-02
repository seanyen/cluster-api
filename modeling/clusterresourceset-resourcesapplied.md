# ClusterResourceSet Reconciler - ResourcesApplied Condition

**Reconciler**: `internal/controllers/clusterresourceset/clusterresourceset_controller.go`  
**Primary Reconciled Object**: `ClusterResourceSet`  
**Condition Type Modeled**: `ResourcesApplied`

## Overview

The `ResourcesApplied` condition on a ClusterResourceSet indicates whether the resources defined in the ClusterResourceSet have been successfully applied to the target clusters. The ClusterResourceSet applies ConfigMaps and Secrets to workload clusters that match its label selector.

---

## Condition Transition Table — `ResourcesApplied` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `ResourcesApplied`) | Next reconcile (Return) |
|---|---|---|---|
| `isPaused == true` | No-op; ClusterResourceSet is paused | (condition unchanged due to paused state) | `none` |
| `error fetching clusters by selector` | No-op; cannot determine target clusters | `ResourcesApplied=False; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero()` | Remove ClusterResourceSet from all ClusterResourceSetBindings; remove finalizer | (condition unchanged; object being deleted) | `none` |
| `selector.Empty() (nil or empty clusterSelector)` | No-op; no clusters match | (condition unchanged; no clusters selected) | `none` |
| `len(clusters) == 0 (no clusters match selector)` | No-op; no clusters to apply to | (condition unchanged; no target clusters) | `none` |
| `cluster.status.initialization.controlPlaneInitialized == false` | Skip cluster; control plane not ready | (condition reflects partial application if some clusters fail) | `none` |
| `error getting remote client for cluster` | Skip cluster; cannot connect to workload cluster | (condition reflects partial application) | `none` |
| `error applying resource (ConfigMap/Secret not found)` | Log error; continue with other resources | (condition reflects failures if any resources fail) | `none` |
| `error applying resource (invalid Secret type)` | Log error; continue with other resources | (condition reflects failures if any resources fail) | `none` |
| `patching conflict on ClusterResourceSetBinding` | Requeue quickly to retry | (condition unchanged) | `requeueAfter: 100ms` |
| `all resources applied successfully to all clusters && strategy=ApplyOnce` | Create/update ClusterResourceSetBinding; mark resources as applied | `ResourcesApplied=True; Reason=Applied; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `all resources applied successfully to all clusters && strategy=Reconcile` | Create/update ClusterResourceSetBinding; reapply if hash changed | `ResourcesApplied=True; Reason=Applied; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `some resources failed to apply` | Update ClusterResourceSetBinding with partial status | `ResourcesApplied=False; Reason=ApplyFailed; Message="<aggregated error details>"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Strategies**: ClusterResourceSet supports two strategies:
   - **ApplyOnce**: Resources are applied only once per cluster. The ClusterResourceSetBinding tracks which resources have been applied.
   - **Reconcile**: Resources are re-applied when their content changes (hash comparison).

2. **ClusterResourceSetBinding**: For each target Cluster, a ClusterResourceSetBinding is created/updated to track:
   - Which ClusterResourceSets target this cluster
   - Which resources have been applied
   - Hash of applied resources (for Reconcile strategy)
   - Application status for each resource

3. **Resource Types**: ClusterResourceSet can reference:
   - **ConfigMaps**: Applied directly to workload cluster
   - **Secrets**: Must be of type `addons.cluster.x-k8s.io/resource-set` to be applied

4. **Control Plane Dependency**: Resources are only applied to clusters whose control plane is initialized (`status.initialization.controlPlaneInitialized == true`).

5. **Conflict Handling**: When multiple ClusterResourceSets target the same cluster and patch the same ClusterResourceSetBinding, conflicts can occur. The controller handles this by requeueing with a short delay (100ms).

6. **Best Effort Application**: The controller continues applying resources even if some fail, aggregating errors in the condition message.

7. **Finalizer**: The ClusterResourceSet uses a finalizer to clean up ClusterResourceSetBinding entries when deleted.

8. **Watch-Driven**: The reconciler watches:
   - ClusterResourceSet objects
   - Clusters (to apply when new clusters match selector)
   - ConfigMaps and Secrets (to re-apply on changes in Reconcile strategy)
   - ClusterCache (for workload cluster connectivity)
