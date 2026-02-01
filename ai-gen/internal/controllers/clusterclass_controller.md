# ClusterClass Controller

The ClusterClass Controller manages `ClusterClass` resources, validating templates, managing variables, and ensuring external references are properly configured.

## Overview

```mermaid
flowchart TB
    subgraph "ClusterClass Controller"
        R[Reconcile] --> F{Fetch ClusterClass}
        F -->|Not Found| End[Return]
        F -->|Found| P{Paused?}
        P -->|Yes| PC[Set Paused Condition & Return]
        P -->|No| D{Deleting?}
        D -->|Yes| DE[Return - no deletion logic]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph "Reconcile Phases"
        RE[reconcileExternalReferences]
        RV[reconcileVariables]
    end
    
    RN --> RE --> RV
```

## Reconciliation Phases

### 1. reconcileExternalReferences

Ensures all referenced templates are owned by the ClusterClass and checks for outdated API versions.

```mermaid
flowchart TD
    A[Start] --> B[Collect All Template Refs]
    B --> C[For Each Unique Ref]
    C --> D[Get Template Object]
    D -->|Not Found| E[Add Error]
    D -->|Found| F[Set Owner Reference]
    F --> G[Check API Version]
    G --> H{Outdated?}
    H -->|Yes| I[Add to Outdated List]
    H -->|No| J[Continue]
    I --> K[Set RefVersionsUpToDate Condition]
```

### 2. reconcileVariables

Processes inline variables and discovers variables from runtime extensions.

```mermaid
flowchart TD
    A[Start] --> B[Process Inline Variables]
    B --> C{RuntimeSDK Enabled?}
    C -->|No| D[Use Inline Only]
    C -->|Yes| E[For Each Patch]
    E --> F{Has DiscoverVariables Extension?}
    F -->|No| G[Skip]
    F -->|Yes| H[Call DiscoverVariables Hook]
    H -->|Error| I[Record Error]
    H -->|Success| J[Merge Variables]
    J --> K[Validate Variables]
    K --> L[Update Status]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| New ClusterClass | Templates defined | Initial creation | Get all templates, set owner refs | VariablesReconciled condition set |
| RefVersionsUpToDate=Unknown | Template refs defined | Templates found | Check API versions for each ref | RefVersionsUpToDate=True/False |
| RefVersionsUpToDate=False | Template refs with old API | Outdated refs detected | Log outdated refs | RefVersionsUpToDate=False with message |
| VariablesReady=Unknown | Inline variables defined | Variables to process | Validate inline variables | VariablesReady reflects validation |
| VariablesReady=Unknown | External patches defined | RuntimeSDK enabled | Call DiscoverVariables hooks | VariablesReady reflects discovery |
| Variables merged | All sources processed | Discovery complete | Validate all variables | VariablesReady=True if valid |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Template not found | Template ref defined | Template missing | Log error, set condition | VariablesReconciled=False |
| DiscoverVariables failed | External patch defined | Extension error | Log error, continue | VariablesReconciled=False |
| Variable validation failed | Variables defined | Invalid schema | Set error condition | VariablesReady=False |
| Any | - | Generic API error | Requeue with error | Error logged, requeue |

## Template References

ClusterClass manages multiple template types:

```mermaid
flowchart TD
    CC[ClusterClass] --> IT[Infrastructure Template]
    CC --> CPT[ControlPlane Template]
    CC --> CPMI[ControlPlane MachineInfrastructure]
    CC --> MDC[MachineDeployment Classes]
    CC --> MPC[MachinePool Classes]
    
    MDC --> MDI[MD Infrastructure Template]
    MDC --> MDB[MD Bootstrap Template]
    
    MPC --> MPI[MP Infrastructure Template]
    MPC --> MPB[MP Bootstrap Template]
```

## Variable Processing

### Inline Variables

```yaml
spec:
  variables:
    - name: imageRepository
      required: true
      schema:
        openAPIV3Schema:
          type: string
          default: "registry.k8s.io"
    - name: workerCount
      required: false
      schema:
        openAPIV3Schema:
          type: integer
          minimum: 1
          maximum: 100
```

### External Variable Discovery

```mermaid
sequenceDiagram
    participant CC as ClusterClass Controller
    participant Cache as Discovery Cache
    participant Ext as Runtime Extension
    
    CC->>Cache: Check Cache
    Cache-->>CC: Cache Miss
    CC->>Ext: DiscoverVariables Request
    Ext-->>CC: Variable Definitions
    CC->>Cache: Store Result
    CC->>CC: Merge with Inline Variables
```

## Status Fields

| Field | Description |
|-------|-------------|
| `status.variables` | List of all discovered variables with definitions |

## Conditions

| Condition | Description |
|-----------|-------------|
| `RefVersionsUpToDate` | Whether all template refs use current API versions |
| `VariablesReady` | Whether variables are valid and ready |
| `Paused` | Set when ClusterClass is paused |

### Variable Status Structure

```yaml
status:
  variables:
    - name: imageRepository
      definitions:
        - from: inline
          required: true
          schema:
            openAPIV3Schema:
              type: string
              default: "registry.k8s.io"
    - name: customVar
      definitions:
        - from: external-patch-name
          required: false
          schema:
            openAPIV3Schema:
              type: object
```

## Owner Reference Management

```mermaid
flowchart TD
    A[ClusterClass] -->|owns| B[InfrastructureTemplate]
    A -->|owns| C[ControlPlaneTemplate]
    A -->|owns| D[BootstrapTemplate]
    A -->|owns| E[MachineInfrastructureTemplate]
    
    subgraph "Ownership Benefits"
        F[Garbage Collection]
        G[clusterctl Move]
        H[Resource Tracking]
    end
```

## API Version Check

The controller checks if template references are using the latest API version:

```mermaid
flowchart TD
    A[Get Template Ref] --> B[Get Current API Version]
    B --> C[Get Contract Version]
    C --> D{Versions Match?}
    D -->|Yes| E[Ref Up-to-Date]
    D -->|No| F[Ref Outdated]
    F --> G[Record Outdated Ref]
    G --> H[Set RefVersionsUpToDate=False]
```

## Watches

The ClusterClass controller watches:

1. **ClusterClass** - Primary resource
2. **ExtensionConfig** - For variable discovery changes

---

[← Back to Index](README.md) | [Previous: MachineHealthCheck Controller](machinehealthcheck_controller.md) | [Next: ClusterResourceSet Controller →](clusterresourceset_controller.md)
