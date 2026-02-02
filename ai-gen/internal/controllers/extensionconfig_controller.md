# ExtensionConfig Controller

The ExtensionConfig Controller manages `ExtensionConfig` resources, which configure runtime extensions for the Cluster API Runtime SDK.

## Overview

```mermaid
flowchart TB
    subgraph \"ExtensionConfig Controller\"
        R[Reconcile] --> W{Registry Ready?}
        W -->|No| RQ[Requeue after 10s]
        W -->|Yes| F{Fetch ExtensionConfig}
        F -->|Not Found| RD[Unregister from Registry, Return nil]
        F -->|Error| Err[Return error]
        F -->|Found| D{Deleting?}
        D -->|Yes| RD2[Unregister from Registry, Return nil]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph \"Normal Reconcile\"
        RO{ReadOnly Mode?}
        V[Validate ExtensionConfig]
        RC[reconcileCABundle - inject from Secret if annotated]
        DIS[discoverExtension - call Discovery endpoint]
        REG[Register handlers in Registry]
    end
    
    RN --> RO
    RO -->|Yes| V --> REG
    RO -->|No| RC --> DIS --> REG
```

## Warmup Process

The controller has a warmup phase to ensure extensions are discovered before other controllers start reconciling:

```mermaid
sequenceDiagram
    participant Manager as Controller Manager
    participant Warmup as WarmupRunnable
    participant Registry as Runtime Registry
    participant EC as ExtensionConfig Controller
    
    Manager->>Warmup: Start
    Warmup->>Warmup: List ExtensionConfigs
    Warmup->>Registry: Register Each EC
    Warmup->>Registry: Mark Ready
    Manager->>EC: Start Reconciling
```

## Reconciliation Phases

### 1. Reconcile CA Bundle

When not in ReadOnly mode, the controller can inject CA certificates from Secrets:

```mermaid
flowchart TD
    A[Start] --> B{InjectCAFromSecretAnnotation?}
    B -->|No| C[Use Provided CABundle]
    B -->|Yes| D[Get Referenced Secret]
    D -->|Not Found| E[Set Error Condition]
    D -->|Found| F[Read ca.crt from Secret]
    F --> G[Set CABundle on ExtensionConfig]
```

### 2. Discover Extensions

```mermaid
flowchart TD
    A[Start] --> B[Call Runtime Extension Discovery]
    B --> C{Discovery Successful?}
    C -->|No| D[Set Error Condition]
    C -->|Yes| E[Parse Handler Registrations]
    E --> F[Update Status with Handlers]
    F --> G[Set DiscoverySucceeded Condition]
```

### 3. Register in Registry

```mermaid
flowchart TD
    A[ExtensionConfig Validated] --> B[Register in Runtime Client]
    B --> C{Registration Successful?}
    C -->|No| D[Log Error, Return]
    C -->|Yes| E[Extension Available for Hooks]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Registry not ready | - | Controller startup | Requeue after 10s | Waiting for warmup |
| New ExtensionConfig | ClientConfig defined | Initial creation | Validate config | Validation in progress |
| Valid config | CA injection annotation | Secret exists | Inject CA from Secret | CABundle updated |
| CABundle set | ClientConfig valid | CA ready | Call Discovery endpoint | Handlers discovered |
| Handlers discovered | - | Discovery complete | Register in registry | Extension available |

### CA Bundle Injection

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| No CABundle | inject-ca-from-secret annotation | Secret exists | Read ca.crt, set CABundle | CABundle populated |
| No CABundle | inject-ca-from-secret annotation | Secret not found | Set error condition | Error logged |
| CABundle outdated | Secret changed | Secret updated | Re-read ca.crt, update CABundle | CABundle updated |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes ExtensionConfig | Unregister from registry | Extension unavailable |
| Not found | - | Object deleted | Unregister from registry | Extension removed |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Discovery failed | - | Extension unreachable | Log error, set condition | Discovered=False |
| Invalid handlers | - | Handler validation failed | Log error, set condition | Discovered=False |
| CA Secret not found | inject annotation set | Secret missing | Log error, set condition | Error condition |
| Any | - | Generic API error | Requeue with error | Error logged, requeue |

## ReadOnly Mode

In ReadOnly mode, the controller only validates and registers extensions without modifying them:

```mermaid
flowchart TD
    A[ReadOnly Mode] --> B[Get ExtensionConfig]
    B --> C[Validate Configuration]
    C --> D{Valid?}
    D -->|Yes| E[Register in Registry]
    D -->|No| F[Skip Registration]
```

This mode is useful in multi-tenant environments where ExtensionConfigs are managed externally.

## CA Bundle Injection

The controller supports automatic CA bundle injection from Secrets:

```yaml
apiVersion: runtime.cluster.x-k8s.io/v1beta2
kind: ExtensionConfig
metadata:
  name: my-extension
  annotations:
    runtime.cluster.x-k8s.io/inject-ca-from-secret: my-namespace/my-ca-secret
spec:
  clientConfig:
    service:
      name: my-extension-service
      namespace: my-namespace
    # caBundle will be automatically populated
```

```mermaid
flowchart TD
    A[ExtensionConfig] --> B[Annotation: inject-ca-from-secret]
    B --> C[Secret Reference]
    C --> D[Read ca.crt Key]
    D --> E[Set clientConfig.caBundle]
```

## Extension Discovery

When discovering extensions, the controller calls the extension's Discovery endpoint:

```mermaid
sequenceDiagram
    participant EC as ExtensionConfig Controller
    participant Ext as Runtime Extension
    
    EC->>Ext: POST /discovery
    Ext-->>EC: DiscoveryResponse
    Note over EC,Ext: Response includes:<br/>- Handler registrations<br/>- Supported hooks
    EC->>EC: Update status.handlers
```

## Status Fields

| Field | Description |
|-------|-------------|
| `status.handlers` | List of discovered extension handlers |

### Handler Structure

```yaml
status:
  handlers:
    - name: before-cluster-create
      hookName: BeforeClusterCreate
      failurePolicy: Fail
      timeoutSeconds: 10
```

## Conditions

| Condition | Description |
|-----------|-------------|
| `Discovered` | Whether extension discovery succeeded |
| `Paused` | Set when ExtensionConfig is paused |

## ExtensionConfig Spec

```yaml
apiVersion: runtime.cluster.x-k8s.io/v1beta2
kind: ExtensionConfig
metadata:
  name: my-extension
spec:
  clientConfig:
    service:
      name: my-extension-webhook
      namespace: my-namespace
      port: 443
      path: /
    caBundle: <base64-encoded-ca-cert>
  namespaceSelector:
    matchLabels:
      environment: production
  settings:
    key1: value1
```

## Registry Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Warming: Controller Start
    Warming --> Ready: Warmup Complete
    Ready --> Reconciling: Normal Operation
    
    state Reconciling {
        [*] --> Registered: ExtensionConfig Created
        Registered --> Updated: ExtensionConfig Changed
        Updated --> Registered: Re-registered
        Registered --> Unregistered: ExtensionConfig Deleted
        Unregistered --> [*]
    }
```

## Watches

The ExtensionConfig controller watches:

1. **ExtensionConfig** - Primary resource
2. **Secret** - For CA bundle injection (when not in ReadOnly mode)

---

[← Back to Index](README.md) | [Previous: ClusterResourceSet Controller](clusterresourceset_controller.md)
