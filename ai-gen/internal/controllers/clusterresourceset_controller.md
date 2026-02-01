# ClusterResourceSet Controller

The ClusterResourceSet Controller manages `ClusterResourceSet` resources, applying ConfigMaps and Secrets to matching clusters based on label selectors.

## Overview

```mermaid
flowchart TB
    subgraph "ClusterResourceSet Controller"
        R[Reconcile] --> F{Fetch CRS}
        F -->|Not Found| End[Return]
        F -->|Found| Fin[Add Finalizer]
        Fin --> P{Paused?}
        P -->|Yes| PC[Set Paused Condition]
        P -->|No| D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph "Reconcile Flow"
        GC[Get Matching Clusters]
        AC[Apply to Each Cluster]
        UB[Update Binding]
    end
    
    subgraph "Delete Flow"
        RB[Remove from Bindings]
        RF[Remove Finalizer]
    end
    
    RN --> GC --> AC --> UB
    RD --> RB --> RF
```

## Resource Application Flow

```mermaid
flowchart TD
    A[Get ClusterResourceSet] --> B[Get Matching Clusters]
    B --> C[For Each Cluster]
    C --> D{Cluster Control Plane Initialized?}
    D -->|No| E[Skip - Wait for CP]
    D -->|Yes| F[Get/Create ClusterResourceSetBinding]
    F --> G[For Each Resource in CRS]
    G --> H{Resource Type}
    H -->|ConfigMap| I[Get ConfigMap]
    H -->|Secret| J[Get Secret]
    I --> K[Parse YAML Content]
    J --> K
    K --> L[Apply to Workload Cluster]
    L --> M[Update Binding Status]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| New CRS | ClusterSelector defined | Initial creation | Find matching clusters | ResourcesApplied=Unknown |
| Matching clusters found | Resources defined | Clusters match selector | Apply resources to clusters | ResourcesApplied in progress |
| Resources applied | strategy=ApplyOnce | First application | Record in binding, skip future applies | ResourcesApplied=True |
| Resources applied | strategy=Reconcile | Content changed | Re-apply resources | ResourcesApplied=True |
| No matching clusters | ClusterSelector defined | No clusters match | Set ResourcesApplied=True (vacuously) | ResourcesApplied=True |

### Resource Application

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Resource not applied | ApplyOnce strategy | New cluster matches | Apply resource, record hash | Binding updated |
| Resource applied | ApplyOnce strategy | Resource changed | Skip re-application | No change |
| Resource applied | Reconcile strategy | Resource changed | Re-apply resource | Binding hash updated |
| Resource missing | Resource defined | ConfigMap/Secret not found | Log error, continue | ResourcesApplied=False |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes CRS | Remove from all bindings | Bindings updated |
| Binding has other CRS | - | CRS removed from binding | Patch binding, remove CRS entry | Binding persists |
| Binding only has this CRS | - | CRS removed from binding | Delete binding | Binding deleted |
| All bindings processed | - | Cleanup complete | Remove finalizer | Object deleted by GC |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Cluster selector invalid | - | Invalid selector | Log error, set condition | ResourcesApplied=False |
| Resource not found | Resource defined | ConfigMap/Secret missing | Log warning, continue | Partial application |
| Resource parse error | Resource defined | Invalid YAML | Log error, continue | Partial application |
| Apply conflict | - | Concurrent binding update | Requeue after 100ms | Retry application |
| Workload cluster unreachable | - | Connection error | Log error, set condition | ResourcesApplied=False |

## Strategy Types

### ApplyOnce Strategy

```mermaid
flowchart TD
    A[Check Binding] --> B{Resource Applied Before?}
    B -->|Yes| C[Skip - Already Applied]
    B -->|No| D[Apply Resource]
    D --> E[Record Hash in Binding]
```

### Reconcile Strategy

```mermaid
flowchart TD
    A[Check Binding] --> B[Compute Resource Hash]
    B --> C{Hash Changed?}
    C -->|Yes| D[Re-Apply Resource]
    C -->|No| E[Skip - No Changes]
    D --> F[Update Hash in Binding]
```

## ClusterResourceSetBinding

The binding tracks which resources have been applied to which clusters:

```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta2
kind: ClusterResourceSetBinding
metadata:
  name: my-cluster  # Same name as cluster
  namespace: default
spec:
  bindings:
    - clusterResourceSetName: my-crs
      resources:
        - applied: true
          hash: "abc123..."
          kind: ConfigMap
          name: my-configmap
        - applied: true
          hash: "def456..."
          kind: Secret
          name: my-secret
```

```mermaid
flowchart TD
    CRS1[ClusterResourceSet 1] --> CRSB[ClusterResourceSetBinding]
    CRS2[ClusterResourceSet 2] --> CRSB
    CRSB --> C[Cluster]
    
    subgraph "Binding Content"
        B1[CRS1 Resources]
        B2[CRS2 Resources]
    end
```

## Resource Processing

### ConfigMap Processing

```mermaid
flowchart TD
    A[Get ConfigMap] --> B[Read 'data' field]
    B --> C[For Each Key-Value]
    C --> D[Parse YAML]
    D --> E[Apply to Workload Cluster]
```

### Secret Processing

```mermaid
flowchart TD
    A[Get Secret] --> B{Secret Type Supported?}
    B -->|No| C[Skip - Unsupported]
    B -->|Yes| D[Read 'data' field]
    D --> E[Decode Base64]
    E --> F[Parse YAML]
    F --> G[Apply to Workload Cluster]
```

Supported Secret types:
- `addons.cluster.x-k8s.io/resource-set` (explicit type)
- `Opaque` (default type)

## Status Fields

ClusterResourceSet doesn't have many status fields, but uses conditions.

## Conditions

| Condition | Description |
|-----------|-------------|
| `ResourcesApplied` | Whether resources have been successfully applied |
| `Paused` | Set when ClusterResourceSet is paused |

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
    B --> C[Find Clusters in Same Namespace]
    C --> D[Filter by Labels]
    D --> E[Exclude Deleting Clusters]
    E --> F[Result: Matching Clusters]
```

## Resource Definition

```yaml
spec:
  strategy: ApplyOnce  # or Reconcile
  resources:
    - name: my-configmap
      kind: ConfigMap
    - name: my-secret
      kind: Secret
```

## Watches

The ClusterResourceSet controller watches:

1. **ClusterResourceSet** - Primary resource
2. **Cluster** - For matching clusters
3. **ConfigMap** - For resource changes
4. **Secret** - For resource changes (partial cache)
5. **ClusterCache** - For workload cluster connection

---

[← Back to Index](README.md) | [Previous: ClusterClass Controller](clusterclass_controller.md) | [Next: ExtensionConfig Controller →](extensionconfig_controller.md)
