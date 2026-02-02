# MachineDeployment Controller

The MachineDeployment Controller manages `MachineDeployment` resources, implementing rollout strategies (RollingUpdate, OnDelete) to update machines in a controlled manner.

## Overview

```mermaid
flowchart TB
    subgraph \"MachineDeployment Controller\"
        R[Reconcile] --> F{Fetch MachineDeployment}
        F -->|Not Found| End[Return nil]
        F -->|Error| Err[Return error]
        F -->|Found| Fin{EnsureFinalizer}
        Fin -->|Added/Error| FinRet[Return]
        Fin -->|Already has| GC[Get Cluster]
        GC -->|Error| GCE[Return error]
        GC --> P{EnsurePausedCondition}
        P -->|Paused/Requeue/Error| PC[Return]
        P -->|Not Paused| D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[reconcile]
    end
    
    subgraph \"Reconcile Flow\"
        GT[getTemplatesAndSetOwner]
        GA[getAndAdoptMachineSetsForDeployment]
        PP{spec.paused?}
        ST{Strategy Type}
        RU[rolloutRollingUpdate]
        OD[rolloutOnDelete]
        SY[sync - scale only, no rollout]
    end
    
    RN --> GT --> GA --> PP
    PP -->|Yes| SY
    PP -->|No| ST
    ST -->|RollingUpdate| RU
    ST -->|OnDelete| OD
```

## Rollout Strategies

### RollingUpdate Strategy

```mermaid
flowchart TD
    A[Start Rollout] --> B[Create Rollout Planner]
    B --> C{Template Changed?}
    C -->|No| D[Scale Existing MS]
    C -->|Yes| E[Create New MachineSet]
    E --> F[Calculate Scale Intents]
    F --> G{Max Surge}
    G --> H[Scale Up New MS]
    H --> I{Max Unavailable}
    I --> J[Scale Down Old MS]
    J --> K{All Replicas Migrated?}
    K -->|No| L[Continue Rollout]
    K -->|Yes| M[Clean Up Old MachineSets]
```

### OnDelete Strategy

```mermaid
flowchart TD
    A[Start] --> B{Template Changed?}
    B -->|No| C[Scale Existing MS]
    B -->|Yes| D[Create New MachineSet]
    D --> E[Wait for User to Delete Machines]
    E --> F{Machine Deleted?}
    F -->|Yes| G[New MS Creates Replacement]
    F -->|No| H[Wait]
    G --> I{All Machines Updated?}
    I -->|No| E
    I -->|Yes| J[Clean Up Old MachineSets]
```

## Reconciliation Details

### Template and MachineSet Management

```mermaid
flowchart TD
    A[Get Templates] --> B{Infra Template Exists?}
    B -->|No| C[Set Condition: NotFound]
    B -->|Yes| D[Set Owner Reference]
    D --> E{Bootstrap Template Exists?}
    E -->|No| F[Set Condition: NotFound]
    E -->|Yes| G[Set Owner Reference]
    G --> H[Get MachineSets]
    H --> I[Filter by Selector]
    I --> J[Adopt Orphaned MachineSets]
```

### Rollout Planner Logic

```mermaid
flowchart TD
    A[Create Planner] --> B[Get Current MS]
    B --> C{Current MS Up-to-Date?}
    C -->|Yes| D[Use Current as New]
    C -->|No| E[Create New MS Spec]
    E --> F[Calculate Revision]
    F --> G[Compute Scale Intents]
    G --> H{Max Surge}
    H --> I[New MS: Current + Surge]
    I --> J{Max Unavailable}
    J --> K[Old MS: Current - Unavailable]
    K --> L[Apply Intents to MachineSets]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation - RollingUpdate

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| No MachineSets | Replicas=3 | Initial creation | Create MachineSet with 3 replicas | RollingOut=False, Replicas=0→3 |
| 1 MS, Replicas=3 | Replicas=5 | User scales up | Update MachineSet replicas to 5 | ScalingUp=True |
| 1 MS, Replicas=5 | Replicas=3 | User scales down | Update MachineSet replicas to 3 | ScalingDown=True |
| 1 MS, Replicas=3 | Template changed | User updates template | Create new MS, start rollout | RollingOut=True |
| Old MS=3, New MS=0 | maxSurge=1 | Rollout started | Scale new MS to 1 | New MS Replicas=1 |
| Old MS=3, New MS=1 | maxUnavailable=1 | New machine ready | Scale old MS to 2 | Old MS Replicas=2 |
| Old MS=0, New MS=3 | - | Rollout complete | Clean up old MachineSet | RollingOut=False |

### Normal Reconciliation - OnDelete

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| 1 MS, Replicas=3 | Template changed | User updates template | Create new MS with 0 replicas | NewMS created |
| Old MS=3, New MS=0 | - | User deletes old machine | Old MS creates replacement in new MS | New MS Replicas=1 |
| Old MS=2, New MS=1 | - | User continues deleting | Continue migration | Progressive update |
| Old MS=0, New MS=3 | - | All machines migrated | Delete old MachineSet | RollingOut=False |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes MD | Delete all owned MachineSets | Deleting=True |
| Has owned MachineSets | - | MachineSets still exist | Wait for MS deletion | Deleting=True |
| No owned MachineSets | - | All MS gone | Remove finalizer | Object deleted by GC |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Template not found | Rollout requested | Template missing | Block rollout, set condition | Available=False |
| MachineSet creation failed | Rollout requested | API error | Log error, requeue | Error logged |
| Any | - | Generic API error | Requeue with error | Error logged, requeue |

## RollingUpdate Parameters

```yaml
spec:
  rollout:
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1          # Max machines above desired during rollout
        maxUnavailable: 0    # Max machines below desired during rollout
        deletePolicy: Oldest # Which machines to delete first
```

### MaxSurge and MaxUnavailable Calculation

```mermaid
flowchart TD
    A[Desired: 3, maxSurge: 1, maxUnavailable: 1] --> B[Max Total: 3 + 1 = 4]
    B --> C[Min Available: 3 - 1 = 2]
    C --> D[Can scale up new MS while old has 3]
    D --> E[When new MS ready, scale down old MS]
```

## Status Fields

| Field | Description |
|-------|-------------|
| `status.replicas` | Total replicas across all MachineSets |
| `status.readyReplicas` | Ready replicas across all MachineSets |
| `status.availableReplicas` | Available replicas across all MachineSets |
| `status.upToDateReplicas` | Replicas matching current template |
| `status.unavailableReplicas` | Unavailable replicas |
| `status.selector` | Label selector in string form |

## Conditions

| Condition | Description |
|-----------|-------------|
| `Available` | Whether minimum available replicas are met |
| `MachinesReady` | Aggregated ready state of all machines |
| `MachinesUpToDate` | Whether all machines match current template |
| `RollingOut` | Whether a rollout is in progress |
| `ScalingUp` | Whether scaling up |
| `ScalingDown` | Whether scaling down |
| `Remediating` | Whether unhealthy machines are being remediated |
| `Deleting` | Set during deletion |

## Revision Management

MachineDeployments track revisions through annotations:

| Annotation | Description |
|------------|-------------|
| `deployment.cluster.x-k8s.io/revision` | Current revision number |
| `deployment.cluster.x-k8s.io/revision-history` | Previous revision numbers |

```mermaid
flowchart TD
    A[Initial Deploy] --> B[Revision 1]
    B --> C[Template Update]
    C --> D[Revision 2]
    D --> E[Another Update]
    E --> F[Revision 3]
    F --> G[Rollback to 1]
    G --> H[Revision 4 with Rev 1 spec]
```

## MachineSet Naming

MachineSets created by MachineDeployment use a hash-based naming scheme:

```
<machinedeployment-name>-<template-hash>
```

The template hash ensures:
- Unique name per template version
- Consistent naming across reconciliations
- Easy identification of current vs old MachineSets

---

[← Back to Index](README.md) | [Previous: MachineSet Controller](machineset_controller.md) | [Next: MachinePool Controller →](machinepool_controller.md)
