# ClusterResourceSet Controller

The ClusterResourceSet Controller manages `ClusterResourceSet` resources, applying ConfigMaps and Secrets to matching clusters based on label selectors. It uses `ClusterResourceSetBinding` to track which resources have been applied to which clusters.

## Overview

```mermaid
flowchart TB
    subgraph "ClusterResourceSet Controller"
        R[Reconcile] --> F{Fetch CRS}
        F -->|Not Found| End[Return nil]
        F -->|Found| Fin[EnsureFinalizer]
        Fin --> P{Paused?}
        P -->|Yes| PC[Set Paused Condition & Return]
        P -->|No| GC[Get Matching Clusters]
        GC --> D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[ApplyClusterResourceSet<br/>for each cluster]
    end
    
    subgraph "Delete Flow"
        RD --> FB[For Each Cluster]
        FB --> GB[Get ClusterResourceSetBinding]
        GB --> RB[Remove CRS from Binding]
        RB --> CB{Binding Empty?}
        CB -->|Yes| DB[Delete Binding]
        CB -->|No| PB[Patch Binding]
        DB --> RF[Remove Finalizer]
        PB --> RF
    end
```

## Resource Application Flow

```mermaid
flowchart TD
    A[ApplyClusterResourceSet] --> B[Get Resources<br/>ConfigMap/Secret]
    B --> C{Resource Found?}
    C -->|No - NotFound| Skip[Continue to next resource]
    C -->|No - Other Error| ErrRes[Add to error list]
    C -->|Yes| D[Ensure OwnerRef on Resource]
    D --> E[Get/Create ClusterResourceSetBinding]
    E --> F[Ensure OwnerRef on Binding]
    F --> G[Get Remote Client via ClusterCache]
    G --> H{Client Available?}
    H -->|No| ErrClient[Set Condition False<br/>Return Error]
    H -->|Yes| I[Ensure K8s Service Created]
    I --> J[For Each Resource]
    J --> K[Create ResourceReconcileScope]
    K --> L{needsApply?}
    L -->|No| M[Skip - Already Applied]
    L -->|Yes| N[Apply to Workload Cluster]
    N --> O{Apply Success?}
    O -->|Yes| P[Set Binding: Applied=true, Hash]
    O -->|No| Q[Set Binding: Applied=false]
    P --> R[Set ResourcesApplied=True]
    Q --> S[Set ResourcesApplied=False]
```

## KRTT - Kubernetes Reconciler Transition Table

### Main Reconcile Loop

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| CRS not found | - | Object deleted before reconcile | Return nil (no-op) | - |
| CRS without finalizer | Any | Object fetched | Add `addons.cluster.x-k8s.io` finalizer | CRS with finalizer |
| CRS paused | Any | Paused annotation or spec | Set Paused condition, skip reconcile | Paused=True |
| Selector invalid | ClusterSelector defined | Invalid label selector | Set ResourcesApplied=False (InternalError) | ResourcesApplied=False |
| Selector empty | ClusterSelector={} | Empty selector | Return nil, no clusters matched | No change |
| DeletionTimestamp!=nil | - | User deletes CRS | Execute `reconcileDelete` | Finalizer removed, GC deletes |
| Matching clusters exist | Resources defined | Normal reconcile | Call `ApplyClusterResourceSet` per cluster | ResourcesApplied condition updated |

### ApplyClusterResourceSet Flow

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Resource (Secret) wrong type | Resource defined | Secret.Type != `addons.cluster.x-k8s.io/resource-set` | Log error, set condition | ResourcesApplied=False (WrongSecretType) |
| Resource not found | Resource defined | ConfigMap/Secret missing | Log warning, continue to next | Continue (partial application) |
| OwnerRef missing on resource | Resource exists | Resource without CRS OwnerRef | Patch resource to add OwnerRef | Resource has OwnerRef |
| Binding not exists | Cluster matched | First CRS for this cluster | Create ClusterResourceSetBinding | Binding created |
| Binding exists | Cluster matched | CRS already bound | Get existing binding | Binding reused |
| Remote client unavailable | - | ClusterCache.GetClient fails | Set ResourcesApplied=False | ResourcesApplied=False (InternalError) |
| K8s service not ready | - | Remote cluster <v1.25 | Wait for `kubernetes` Service in default ns | Continue after service exists |

### Resource Application (per resource)

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Not applied (ApplyOnce) | strategy=ApplyOnce | `IsApplied(ref)` returns false | Create objects in workload cluster | Binding: Applied=true, Hash set |
| Already applied (ApplyOnce) | strategy=ApplyOnce | `IsApplied(ref)` returns true | Skip (needsApply=false) | No change |
| Hash unchanged (Reconcile) | strategy=Reconcile | Hash matches binding hash | Skip (needsApply=false) | No change |
| Hash changed (Reconcile) | strategy=Reconcile | Hash differs from binding hash | Patch objects in workload cluster | Binding: Applied=true, new Hash |
| Object not exists (Reconcile) | strategy=Reconcile | Object not in workload cluster | Create object | Binding updated |
| Apply fails | Any | Create/Patch error | Log error, set Applied=false | ResourcesApplied=False (NotApplied) |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes CRS | For each matching cluster: process binding | Start cleanup |
| Binding not found | - | Binding already deleted | Remove finalizer from CRS | CRS ready for GC |
| Binding has this CRS + others | - | Multiple CRS reference binding | Remove CRS entry, remove OwnerRef, Patch binding | Binding persists |
| Binding only has this CRS | - | Single CRS in binding | Delete entire ClusterResourceSetBinding | Binding deleted |
| All bindings processed | - | Cleanup complete | Remove finalizer from CRS | CRS deleted by GC |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| ClusterSelector parse error | ClusterSelector defined | Invalid selector syntax | Return error, set ResourcesApplied=False | ResourcesApplied=False (InternalError), Requeue with backoff |
| ConfigMap/Secret not found | Resource defined | Missing resource | Log, continue (best effort) | Partial application |
| Secret wrong type | Resource defined | Type != resource-set | Set condition, add to error list | ResourcesApplied=False (WrongSecretType) |
| YAML parse error | Resource defined | Invalid YAML in ConfigMap/Secret | Add error, set Applied=false | ResourcesApplied=False |
| Remote cluster unreachable | - | ClusterCache.GetClient fails | Set condition, return error | ResourcesApplied=False (InternalError), Requeue |
| Binding patch conflict | - | Concurrent update by another CRS | Requeue after 100ms | Retry after fixed interval |
| Apply object fails | Any | Create/Patch error on workload cluster | Log error, continue to next object | Partial application, Applied=false |

## Strategy Types

### ApplyOnce Strategy (Default)

The `ApplyOnce` strategy applies resources only once to a cluster. Even if the ConfigMap/Secret content changes, it will not be re-applied.

```mermaid
flowchart TD
    A[reconcileApplyOnceScope.needsApply] --> B{IsApplied in Binding?}
    B -->|Yes| C[Return false - Skip]
    B -->|No| D[Return true - Apply]
    D --> E[applyObj: Create only]
    E --> F{Object exists?}
    F -->|Yes - AlreadyExists| G[Ignore error, return nil]
    F -->|No| H[Create object]
```

**Key characteristics:**
- Checks `ResourceSetBinding.IsApplied(resourceRef)` - if `Applied=true`, skip
- Uses `createUnstructured()` which only creates, ignores `AlreadyExists` errors
- Hash is stored but never used for comparison in this strategy

### Reconcile Strategy

The `Reconcile` strategy re-applies resources when their content changes, detected via hash comparison.

```mermaid
flowchart TD
    A[reconcileStrategyScope.needsApply] --> B{Resource in Binding?}
    B -->|No| C[Return true - Apply]
    B -->|Yes| D{Applied=true?}
    D -->|No| E[Return true - Apply]
    D -->|Yes| F{Hash matches?}
    F -->|Yes| G[Return false - Skip]
    F -->|No| H[Return true - Apply]
    C --> I[applyObj: Get then Create/Patch]
    E --> I
    H --> I
    I --> J{Object exists?}
    J -->|No| K[Create object]
    J -->|Yes| L[Patch object with MergeFrom]
```

**Key characteristics:**
- Checks: `resourceBinding == nil || !Applied || Hash != computedHash`
- Uses `Get` + `Patch` (MergeFrom) for existing objects
- Updates hash in binding after successful apply

## ClusterResourceSetBinding

The binding tracks which resources have been applied to which clusters. It has the same name as the cluster.

```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta2
kind: ClusterResourceSetBinding
metadata:
  name: my-cluster  # Same name as cluster
  namespace: default
  ownerReferences:
    - apiVersion: addons.cluster.x-k8s.io/v1beta2
      kind: ClusterResourceSet
      name: my-crs
      uid: xxx
spec:
  clusterName: my-cluster
  bindings:
    - clusterResourceSetName: my-crs
      resources:
        - name: my-configmap
          kind: ConfigMap
          applied: true
          hash: "sha256:abc123..."
          lastAppliedTime: "2025-01-15T10:00:00Z"
        - name: my-secret
          kind: Secret
          applied: true
          hash: "sha256:def456..."
          lastAppliedTime: "2025-01-15T10:00:00Z"
```

```mermaid
flowchart TD
    CRS1[ClusterResourceSet 1] --> CRSB[ClusterResourceSetBinding]
    CRS2[ClusterResourceSet 2] --> CRSB
    CRSB --> C[Cluster]
    
    subgraph "Binding Structure"
        direction TB
        B1[Binding for CRS1<br/>Resources: ConfigMap-A, Secret-B]
        B2[Binding for CRS2<br/>Resources: ConfigMap-C]
    end
    
    CRSB --> B1
    CRSB --> B2
```

## Resource Processing

### ConfigMap Processing

```mermaid
flowchart TD
    A[getResource - ConfigMap] --> B[Get ConfigMap from API]
    B --> C[Convert to Unstructured]
    C --> D[normalizeData]
    D --> E[Sort keys alphabetically]
    E --> F[Extract data values as bytes]
    F --> G[objsFromYamlData]
    G --> H{JSON list?}
    H -->|Yes| I[Unmarshal JSON array]
    H -->|No| J[Parse YAML documents]
    I --> K[SortForCreate]
    J --> K
    K --> L[Apply objects to workload cluster]
```

### Secret Processing

```mermaid
flowchart TD
    A[getResource - Secret] --> B[Get Secret from API]
    B --> C{Type Check}
    C -->|addons.cluster.x-k8s.io/resource-set| D[Valid - Convert to Unstructured]
    C -->|Other types| E[Return ErrSecretTypeNotSupported]
    D --> F[normalizeData]
    F --> G[Sort keys alphabetically]
    G --> H[Base64 decode values]
    H --> I[objsFromYamlData]
    I --> J[Parse YAML/JSON]
    J --> K[SortForCreate]
    K --> L[Apply objects to workload cluster]
```

**Note:** Only Secrets with type `addons.cluster.x-k8s.io/resource-set` are supported. `Opaque` type is NOT supported.

## Hash Computation

The hash is computed for change detection in the Reconcile strategy:

```mermaid
flowchart LR
    A[Resource Data] --> B[normalizeData]
    B --> C[Sort keys<br/>alphabetically]
    C --> D[Base64 decode<br/>if Secret]
    D --> E[computeHash]
    E --> F[SHA256 of all<br/>data values]
    F --> G[sha256:hexstring]
```

The hash is deterministic because:
1. Keys are sorted alphabetically
2. Each value is written to the hash in order
3. Format: `sha256:<hex-encoded-hash>`

## Status Fields

### ClusterResourceSet Status

| Field | Description |
|-------|-------------|
| `conditions` | Standard Kubernetes conditions (metav1.Condition) |
| `observedGeneration` | Last reconciled generation |
| `deprecated.v1beta1.conditions` | Legacy v1beta1 conditions (for backward compatibility) |

## Conditions

| Condition | Status | Reason | Description |
|-----------|--------|--------|-------------|
| `ResourcesApplied` | True | `Applied` | All resources successfully applied to all matching clusters |
| `ResourcesApplied` | False | `NotApplied` | Failed to apply at least one resource |
| `ResourcesApplied` | False | `WrongSecretType` | Secret type is not supported |
| `ResourcesApplied` | False | `InternalError` | Unexpected failure (selector parse, client error, etc.) |
| `Paused` | True | - | ClusterResourceSet is paused |

## Cluster Selection

```yaml
spec:
  clusterSelector:
    matchLabels:
      environment: production
    matchExpressions:
      - key: cluster.x-k8s.io/cluster-name
        operator: NotIn
        values: ["excluded-cluster"]
```

```mermaid
flowchart TD
    A[ClusterResourceSet] --> B[ClusterSelector]
    B --> C{Empty Selector?}
    C -->|Yes| D[Match Nothing<br/>Return empty list]
    C -->|No| E[List Clusters in Same Namespace]
    E --> F[Filter by Label Selector]
    F --> G[Exclude Clusters with<br/>DeletionTimestamp]
    G --> H[Result: Matching Clusters]
```

**Important:** An empty `ClusterSelector` matches **nothing**, not everything.

## Resource Definition

```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta2
kind: ClusterResourceSet
metadata:
  name: my-crs
  namespace: default
spec:
  clusterSelector:
    matchLabels:
      cni: calico
  strategy: ApplyOnce  # or Reconcile (default: ApplyOnce)
  resources:
    - name: my-configmap
      kind: ConfigMap
    - name: my-secret
      kind: Secret
```

## Watches

The ClusterResourceSet controller watches:

| Resource | Watch Type | Handler | Purpose |
|----------|------------|---------|---------|
| `ClusterResourceSet` | Primary | Reconcile | Main reconciliation trigger |
| `Cluster` | Watch | `clusterToClusterResourceSet` | Trigger when cluster labels change or new cluster created |
| `ClusterCache` | RawSource | `clusterToClusterResourceSet` | Trigger when workload cluster connection changes |
| `ConfigMap` | WatchesMetadata | `resourceToClusterResourceSetFunc` | Trigger when ConfigMap created/updated |
| `Secret` | RawSource (partial cache) | `resourceToClusterResourceSetFunc` | Trigger when Secret created/updated |

### Mapper Functions

**clusterToClusterResourceSet:**
- Lists all ClusterResourceSets in the same namespace
- For each CRS, checks if cluster labels match the selector
- Returns requests for matching CRS

**resourceToClusterResourceSetFunc:**
- If resource has CRS OwnerReferences, return those CRS names
- Otherwise, list all CRS in namespace and find those referencing this resource

## Finalizer

| Finalizer | Purpose |
|-----------|---------|
| `addons.cluster.x-k8s.io` | Ensures CRS entry is removed from all ClusterResourceSetBindings before deletion |

---

[← Back to Index](README.md) | [Previous: ClusterClass Controller](clusterclass_controller.md) | [Next: ExtensionConfig Controller →](extensionconfig_controller.md)
