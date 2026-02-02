# KubeadmConfig Reconciler - Ready Condition

**Reconciler**: `bootstrap/kubeadm/internal/controllers/kubeadmconfig_controller.go`  
**Primary Reconciled Object**: `KubeadmConfig`  
**Condition Type Modeled**: `Ready`

## Overview

The `Ready` condition on a KubeadmConfig indicates whether the bootstrap configuration is ready for a Machine to consume. This is a summary condition computed from `DataSecretAvailable` and `CertificatesAvailable` conditions.

---

## Condition Transition Table — `Ready` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Ready`) | Next reconcile (Return) |
|---|---|---|---|
| `DataSecretAvailable=True && CertificatesAvailable=True` | No-op; summary condition computed | `Ready=True; Reason=Ready; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `DataSecretAvailable=False && reason=WaitingForClusterInfrastructure` | No-op; waiting for cluster infrastructure | `Ready=False; Reason=NotReady; Message="Waiting for Cluster status.infrastructureReady to be true"; ObservedGeneration=metadata.generation` | `none` |
| `DataSecretAvailable=False && reason=DataSecretNotAvailable` | No-op; data secret not yet created | `Ready=False; Reason=NotReady; Message="<aggregated message from sub-conditions>"; ObservedGeneration=metadata.generation` | `none` |
| `CertificatesAvailable=False` | No-op; certificates not available | `Ready=False; Reason=NotReady; Message="<aggregated message from sub-conditions>"; ObservedGeneration=metadata.generation` | `none` |
| `DataSecretAvailable=Unknown || CertificatesAvailable=Unknown` | No-op; unknown status from sub-conditions | `Ready=Unknown; Reason=ReadyUnknown; Message="<aggregated message from sub-conditions>"; ObservedGeneration=metadata.generation` | `none` |

---

## Sub-Condition: DataSecretAvailable

| Guard (Observed predicate) | Condition transition |
|---|---|
| `!cluster.status.initialization.infrastructureProvisioned` | `DataSecretAvailable=False; Reason=DataSecretNotAvailable; Message="Waiting for Cluster status.infrastructureReady to be true"` |
| `configOwner.DataSecretName() != nil && status not updated` | `DataSecretAvailable=True; Reason=DataSecretAvailable` (status recovery after pivot) |
| `status.initialization.dataSecretCreated == true` | `DataSecretAvailable=True; Reason=DataSecretAvailable` |
| Bootstrap secret created successfully | `DataSecretAvailable=True; Reason=DataSecretAvailable` |
| Error creating bootstrap secret | `DataSecretAvailable=False; Reason=DataSecretNotAvailable; Message="<error details>"` |

---

## Sub-Condition: CertificatesAvailable

| Guard (Observed predicate) | Condition transition |
|---|---|
| Certificates exist and are valid | `CertificatesAvailable=True; Reason=CertificatesAvailable` |
| Certificates missing or invalid | `CertificatesAvailable=False; Reason=CertificatesNotAvailable; Message="<details>"` |
| Error checking certificates | `CertificatesAvailable=Unknown; Reason=CertificatesAvailableUnknown` |

---

## Notes

1. **Summary Condition**: The `Ready` condition is computed via `conditions.SetSummaryCondition()` which aggregates:
   - `DataSecretAvailable`
   - `CertificatesAvailable`

2. **Token TTL**: Bootstrap tokens have a configurable TTL (default 15 minutes). The controller handles token refresh.

3. **Control Plane Init Lock**: For the first control plane machine, the controller uses an init lock to ensure only one machine performs kubeadm init.

4. **Watch-Driven**: The reconciler is watch-driven and watches:
   - Machines (owner reference)
   - MachinePools (if feature enabled)
   - Clusters (for infrastructure readiness)
   - ClusterCache source (for remote cluster connection)

5. **Deletion Handling**: Deleted KubeadmConfigs are ignored (no finalizer cleanup needed for bootstrap configs).
