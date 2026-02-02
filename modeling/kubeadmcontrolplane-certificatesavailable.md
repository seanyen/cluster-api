# KubeadmControlPlane Reconciler - CertificatesAvailable Condition

**Reconciler**: `controlplane/kubeadm/internal/controllers/controller.go`  
**Primary Reconciled Object**: `KubeadmControlPlane`  
**Condition Type Modeled**: `CertificatesAvailable`

## Overview

The `CertificatesAvailable` condition on a KubeadmControlPlane indicates whether all required control plane certificates are present and valid. This condition is set during the certificate reconciliation phase.

---

## Condition Transition Table — `CertificatesAvailable` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `CertificatesAvailable`) | Next reconcile (Return) |
|---|---|---|---|
| `all required certificates exist && valid` | No-op; certificates are available | `CertificatesAvailable=True; Reason=CertificatesAvailable; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `any required certificate missing` | Attempt to generate/lookup missing certificates | `CertificatesAvailable=False; Reason=CertificatesNotAvailable; Message="Missing certificates: <list>"; ObservedGeneration=metadata.generation` | `requeue` |
| `certificate generation failed` | Surface error | `CertificatesAvailable=False; Reason=CertificatesNotAvailable; Message="Failed to generate certificates: <error>"; ObservedGeneration=metadata.generation` | `requeue` |
| `certificate lookup/validation failed` | Surface error | `CertificatesAvailable=Unknown; Reason=CertificatesInspectionFailed; Message="Failed to inspect certificates: <error>"; ObservedGeneration=metadata.generation` | `requeue` |

---

## Notes

1. **Required Certificates**: The condition checks for presence and validity of:
   - CA certificate and key
   - API server certificate and key
   - API server kubelet client certificate and key
   - Front proxy CA certificate and key
   - Front proxy client certificate and key
   - etcd CA certificate and key (if etcd is managed)
   - etcd server certificate and key (if etcd is managed)
   - SA (Service Account) key pair

2. **Certificate Storage**: Certificates are stored in Kubernetes Secrets in the same namespace as the KubeadmControlPlane.

3. **Impact on Available**: The `CertificatesAvailable` condition is used by the `Available` condition - if certificates are not available, the control plane is not considered available.

4. **Watch-Driven**: The reconciler is watch-driven; it watches certificate Secrets and reconciles when they change.
