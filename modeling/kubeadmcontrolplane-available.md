# KubeadmControlPlane Reconciler - Available Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `Available`

## Overview

The `Available` condition on a KubeadmControlPlane indicates whether the control plane is available to serve requests. This condition considers the initialization status, etcd cluster health (if managed), Kubernetes control plane component health, and certificate availability.

---

## Condition Transition Table — `Available` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Available`) | Next reconcile (Return) |
|---|---|---|---|
| `status.initialization.controlPlaneInitialized == false` | No-op; control plane not yet initialized | `Available=False; Reason=NotAvailable; Message="Control plane not yet initialized"; ObservedGeneration=metadata.generation` | `requeueAfter: 20s` |
| `etcdIsManaged && etcdMembers == nil && time.Since(Initialized.LastTransitionTime) < 2m` | No-op; wait for etcd to report members | `Available=False; Reason=NotAvailable; Message="Waiting for etcd to report the list of members"; ObservedGeneration=metadata.generation` | `requeueAfter: 20s` |
| `etcdIsManaged && etcdMembers == nil && time.Since(Initialized.LastTransitionTime) >= 2m` | No-op; failed to get etcd members | `Available=Unknown; Reason=AvailableInspectionFailed; Message="Failed to get etcd members"; ObservedGeneration=metadata.generation` | `requeueAfter: 20s` |
| `etcdIsManaged && !etcdMembersAndMachinesAreMatching` | No-op; etcd members don't match machines | `Available=False; Reason=NotAvailable; Message="The list of etcd members does not match the list of Machines and Nodes"; ObservedGeneration=metadata.generation` | `none` |
| `!deletionTimestamp.IsZero()` | No-op; control plane is being deleted | `Available=False; Reason=NotAvailable; Message="* Control plane metadata.deletionTimestamp is set"; ObservedGeneration=metadata.generation` | `none` |
| `CertificatesAvailable != True` | No-op; certificates not available | `Available=False; Reason=NotAvailable; Message="* Control plane certificates are not available"; ObservedGeneration=metadata.generation` | `none` |
| `etcdIsManaged && etcdMembersHealthy < etcdQuorum` | No-op; insufficient healthy etcd members | `Available=False; Reason=NotAvailable; Message="* <N> of <M> etcd members are healthy, at least <Q> healthy member required for etcd quorum"; ObservedGeneration=metadata.generation` | `none` |
| `k8sControlPlaneHealthy < 1` | No-op; no healthy K8s control plane | `Available=False; Reason=NotAvailable; Message="* There are no Machines with healthy control plane components, at least 1 required"; ObservedGeneration=metadata.generation` | `none` |
| `(!etcdIsManaged || etcdMembersHealthy >= etcdQuorum) && k8sControlPlaneHealthy >= 1 && CertificatesAvailable=True && some components not healthy` | No-op; available but with degraded components | `Available=True; Reason=Available; Message="* <N> of <M> etcd members are healthy...\n* <X> of <Y> Machines have healthy control plane components..."; ObservedGeneration=metadata.generation` | `none` |
| `(!etcdIsManaged || etcdMembersHealthy >= etcdQuorum) && k8sControlPlaneHealthy >= 1 && CertificatesAvailable=True && all components healthy` | No-op; fully available | `Available=True; Reason=Available; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Etcd Quorum**: The etcd quorum is calculated as `(votingMembers / 2) + 1`. Learner members do not count toward quorum.

2. **K8s Control Plane Health**: For each Machine with a provider ID, the controller checks:
   - `APIServerPodHealthy` condition
   - `ControllerManagerPodHealthy` condition
   - `SchedulerPodHealthy` condition
   - `EtcdMemberHealthy` condition (if etcd is managed)
   - `EtcdPodHealthy` condition (if etcd is managed)

3. **Surface Delay for Flapping**: When `Available=True`, failures are only surfaced after 10 seconds to avoid condition flapping.

4. **Requeue for Initialization**: The controller requeues with 20s interval while waiting for:
   - Control plane initialization
   - Control plane components to become healthy

5. **Dependencies**:
   - `CertificatesAvailable` condition must be True
   - `Initialized` condition (status.initialization.controlPlaneInitialized)
   - Machine conditions for component health
