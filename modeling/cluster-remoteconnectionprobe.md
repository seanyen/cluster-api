# Cluster Reconciler - RemoteConnectionProbe Condition

**Reconciler**: `internal/controllers/cluster/cluster_controller.go`  
**Primary Reconciled Object**: `Cluster`  
**Condition Type Modeled**: `RemoteConnectionProbe`

## Overview

The `RemoteConnectionProbe` condition on a Cluster reflects the health of the connection to the workload cluster's API server. It is managed by the ClusterCache component and surfaced by the Cluster reconciler.

---

## Condition Transition Table — `RemoteConnectionProbe` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RemoteConnectionProbe`) | Next reconcile (Return) |
|---|---|---|---|
| `healthCheckingState.LastProbeSuccessTime.IsZero() && healthCheckingState.ConsecutiveFailures < 5 && !conditions.Has(RemoteConnectionProbe)` | Set initial condition; connection not yet established | `RemoteConnectionProbe=False; Reason=ProbeFailed; Message="Remote connection not established yet"; ObservedGeneration=metadata.generation` | `none` |
| `healthCheckingState.LastProbeSuccessTime.IsZero() && healthCheckingState.ConsecutiveFailures < 5 && conditions.Has(RemoteConnectionProbe)` | No-op; preserve existing condition during startup | (condition unchanged) | `none` |
| `time.Since(healthCheckingState.LastProbeSuccessTime) > remoteConnectionGracePeriod && healthCheckingState.LastProbeSuccessTime.IsZero()` | Surface probe failure; never succeeded | `RemoteConnectionProbe=False; Reason=ProbeFailed; Message="Remote connection probe failed"; ObservedGeneration=metadata.generation` | `none` |
| `time.Since(healthCheckingState.LastProbeSuccessTime) > remoteConnectionGracePeriod && !healthCheckingState.LastProbeSuccessTime.IsZero()` | Surface probe failure with last success timestamp | `RemoteConnectionProbe=False; Reason=ProbeFailed; Message="Remote connection probe failed, probe last succeeded at <timestamp>"; ObservedGeneration=metadata.generation` | `none` |
| `time.Since(healthCheckingState.LastProbeSuccessTime) <= remoteConnectionGracePeriod` | Probe succeeded within grace period | `RemoteConnectionProbe=True; Reason=ProbeSucceeded; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Health Checking State**: The condition is derived from `ClusterCache.GetHealthCheckingState()` which tracks:
   - `LastProbeSuccessTime`: Timestamp of the last successful health probe
   - `ConsecutiveFailures`: Number of consecutive probe failures

2. **Grace Period**: The `remoteConnectionGracePeriod` (configurable, default 40s) allows for temporary network issues without immediately marking the connection as failed.

3. **Startup Behavior**: During controller startup or when a new Cluster is created, the condition is not updated until at least 5 consecutive failures occur, preventing false negatives.

4. **Watch-Driven**: The reconciler is watch-driven via the ClusterCache source; no explicit `requeueAfter` is used for this condition.

5. **Impact on Available**: This condition is included in the summary computation for the `Available` condition.
