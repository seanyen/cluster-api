# Machine Controller

The Machine Controller manages the lifecycle of individual `Machine` resources, coordinating bootstrap configuration, infrastructure provisioning, node association, and machine deletion with proper cleanup.

## Overview

```mermaid
flowchart TB
    subgraph "Machine Controller"
        R[Reconcile] --> F{Fetch Machine}
        F -->|Not Found| End[Return nil]
        F -->|Error| Err[Return error]
        F -->|Found| Fin{EnsureFinalizer}
        Fin -->|Added/Error| FinRet[Return]
        Fin -->|Already has| LOG[AddOwners to log context]
        LOG --> GC[Get Cluster]
        GC -->|Error| GCE[Return error]
        GC --> P{EnsurePausedCondition}
        P -->|Paused/Requeue/Error| PC[Return]
        P -->|Not Paused| OWN[Get owningMachineSet/MachineDeployment]
        OWN --> D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph "Always Reconcile Phases (both delete and normal)"
        OL[reconcileMachineOwnerAndLabels]
        B[reconcileBootstrap]
        I[reconcileInfrastructure]
        N[reconcileNode]
        CE[reconcileCertificateExpiry]
    end
    
    subgraph "Normal Only"
        IPU[reconcileInPlaceUpdate]
    end
    
    subgraph "Delete Phases (after always phases)"
        IDA[isDeleteNodeAllowed?]
        PreDrain[Pre-Drain Hook check]
        Drain[Drain Node]
        VolumeDetach[Wait Volume Detach]
        PreTerm[Pre-Terminate Hook check]
        DelInfra[Delete Infrastructure]
        DelBoot[Delete Bootstrap]
        DelNode[Delete Node]
        RemFin[Remove Finalizer]
    end
    
    subgraph "Defer: updateStatus + patchMachine"
        US[updateStatus: phase, conditions]
        PM[patchMachine with owned conditions]
    end
    
    RN --> OL --> B --> I --> N --> CE --> IPU
    RD --> OL --> B --> I --> N --> CE --> IDA --> PreDrain --> Drain --> VolumeDetach --> PreTerm --> DelInfra --> DelBoot --> DelNode --> RemFin
```

## Reconciliation Phases

### 1. reconcileBootstrap

Reconciles the `spec.bootstrap.configRef` object.

```mermaid
flowchart TD
    A[Start] --> B{ConfigRef Defined?}
    B -->|No| End[Return - user provides data secret]
    B -->|Yes| C[Get External Object]
    C -->|Not Found| D{Deleting?}
    D -->|Yes| E[Tolerate - Return]
    D -->|No| F[Requeue after 30s]
    C -->|Found| G{DataSecretName set on Machine?}
    G -->|Yes| H[Mark BootstrapDataSecretCreated=true]
    G -->|No| I{Bootstrap reports dataSecretCreated?}
    I -->|No| J[Wait for data secret]
    I -->|Yes| K[Get dataSecretName from Bootstrap]
    K --> L[Set spec.bootstrap.dataSecretName]
    L --> M[Mark BootstrapDataSecretCreated=true]
```

### 2. reconcileInfrastructure

Reconciles the `spec.infrastructureRef` object.

```mermaid
flowchart TD
    A[Start] --> B[Get External Object]
    B -->|Not Found| C{Deleting?}
    C -->|Yes| D[Tolerate - Return]
    C -->|No| E{Was Provisioned?}
    E -->|Yes| F[Error: Deleted after provisioned]
    E -->|No| G[Requeue after 30s]
    B -->|Found| H{Provisioned?}
    H -->|No| I[Wait for provisioning]
    H -->|Yes| J{ProviderID set?}
    J -->|No| K[Wait for ProviderID]
    J -->|Yes| L[Copy ProviderID to Machine]
    L --> M[Copy Addresses]
    M --> N[Copy FailureDomain]
    N --> O[Mark InfrastructureProvisioned=true]
```

### 3. reconcileNode

Associates the Machine with a Kubernetes Node.

```mermaid
flowchart TD
    A[Start] --> B{ProviderID set?}
    B -->|No| End[Return]
    B -->|Yes| C{NodeRef set?}
    C -->|Yes| D[Get Node by Name]
    C -->|No| E[Find Node by ProviderID]
    E -->|Not Found| F{Timeout?}
    F -->|Yes| G[Mark NodeHealthy=Unknown]
    F -->|No| H[Requeue]
    E -->|Found| I[Set NodeRef]
    D -->|Found| J[Sync Labels/Annotations]
    J --> K[Update Node Conditions]
    D -->|Not Found| L{Deleting?}
    L -->|Yes| M[Continue deletion]
    L -->|No| N[Error: Node deleted]
```

### 4. reconcileDelete

Handles machine deletion with ordered cleanup.

```mermaid
flowchart TD
    A[Start] --> B[Set fallback: deletingReason=Reason, deletingMessage=Deletion started]
    B --> C{isDeleteNodeAllowed?}
    C -->|No - errNoControlPlaneNodes| Skip[Skip node operations]
    C -->|No - errLastControlPlaneNode| Skip
    C -->|No - errNilNodeRef| Skip
    C -->|No - errClusterIsBeingDeleted| Skip
    C -->|No - errControlPlaneIsBeingDeleted| Skip
    C -->|Other Error| InternalErr[deletingReason: InternalError<br/>Return error]
    C -->|Yes| D{Pre-Drain Hook annotations?}
    D -->|Yes| E[deletingReason: WaitingForPreDrainHook<br/>Return - wait for annotation removal]
    D -->|No| F{isNodeDrainAllowed?}
    F -->|No| G[Skip drain]
    F -->|Yes| H[Set NodeDrainStartTime]
    H --> I[drainNode]
    I -->|In Progress| J[deletingReason: DrainingNode<br/>Requeue after 20s]
    I -->|Complete| K{isNodeVolumeDetachAllowed?}
    I -->|Error| DrainErr[deletingReason: DrainingNode<br/>Return error]
    G --> K
    K -->|No| L[Skip volume detach]
    K -->|Yes| M[Set WaitForNodeVolumeDetachStartTime]
    M --> N[waitForVolumeDetach]
    N -->|In Progress| O[deletingReason: WaitingForVolumeDetach<br/>Requeue after 20s]
    N -->|Complete| P{Pre-Terminate Hook annotations?}
    N -->|Timeout| P
    L --> P
    P -->|Yes| Q[deletingReason: WaitingForPreTerminateHook<br/>Return - wait for annotation removal]
    P -->|No| Skip
    Skip --> R[Delete InfraMachine]
    R -->|In Progress| S[deletingReason: WaitingForInfrastructureDeletion<br/>Return]
    R -->|Complete| T[Delete BootstrapConfig]
    T -->|In Progress| U[deletingReason: WaitingForBootstrapDeletion<br/>Return]
    T -->|Complete| V{isDeleteNodeAllowed & NodeRef set?}
    V -->|Yes| W[deleteNode]
    V -->|No| X[Skip node deletion]
    W --> X
    X --> Y[Remove Finalizer]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Machine not found | - | Object deleted | Return nil (no-op) | - |
| Machine without finalizer | Any | Object fetched | Add `machine.cluster.x-k8s.io` finalizer, return | Machine with finalizer |
| Machine paused | Any | Paused annotation or Cluster paused | `EnsurePausedCondition` sets condition | Paused=True |
| Stand-alone machine | No MachineSet owner | First reconcile | Set Cluster as owner reference | Machine has Cluster owner |
| Phase=Pending | Bootstrap.ConfigRef defined | Initial creation | Get bootstrap object, set owner ref, watch | BootstrapConfigReady mirrors bootstrap |
| BootstrapReady=False | ConfigRef defined | Bootstrap created | Wait for dataSecretCreated | BootstrapReady=False |
| BootstrapReady=False | ConfigRef defined | dataSecretCreated=true on bootstrap | Get dataSecretName, set on Machine.spec | BootstrapDataSecretCreated=True |
| InfrastructureReady=False | InfraRef defined | Bootstrap ready | Get infra object, set owner ref, watch | InfrastructureReady mirrors infra |
| InfrastructureReady=False | InfraRef defined | Infra provisioned=true | Wait for ProviderID and addresses | InfrastructureReady=False |
| InfrastructureReady=False | InfraRef defined | ProviderID set | Copy ProviderID, addresses, failureDomain | InfrastructureProvisioned=True |
| NodeRef=nil | ProviderID set | Infra ready | Find node by ProviderID via ClusterCache | NodeRef set if found |
| NodeRef set | - | Node exists | Sync labels/annotations to node | NodeReady/NodeHealthy updated |
| Phase=Provisioning | All ready | Node associated | setPhase in updateStatus | Phase=Running |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes Machine | Run alwaysReconcile phases first, then reconcileDelete | Deleting=True (default reason) |
| isDeleteNodeAllowed=false | - | errNoControlPlaneNodes/errLastControlPlaneNode/etc | Skip node operations (drain, volume detach, node delete) | Continue to infra delete |
| Has pre-drain hook annotation | - | Annotation with prefix `pre-drain.delete.hook.machine` | Wait for hook owner to remove annotation | Deleting reason=WaitingForPreDrainHook |
| No pre-drain hooks | NodeRef set | isNodeDrainAllowed=true | Set NodeDrainStartTime, start node drain | DrainingSucceeded=False |
| Draining in progress | - | Pods being evicted | Continue draining, handle timeouts per MachineDrainRules | Deleting reason=DrainingNode |
| Draining complete | - | All pods evicted/skipped | Mark DrainingSucceeded=True | DrainingSucceeded=True |
| isNodeVolumeDetachAllowed=true | - | Drain complete | Set WaitForNodeVolumeDetachStartTime, wait for volumes | Deleting reason=WaitingForVolumeDetach |
| Waiting for volumes | - | CSI volumes still attached | Wait for volume detachment with timeout | Deleting reason=WaitingForVolumeDetach |
| Volumes detached or timeout | - | All volumes detached OR timeout exceeded | Check pre-terminate hooks | VolumeDetachSucceeded=True |
| Has pre-terminate hook annotation | - | Annotation with prefix `pre-terminate.delete.hook.machine` | Wait for hook owner to remove annotation | Deleting reason=WaitingForPreTerminateHook |
| No pre-terminate hooks | InfraRef exists | Pre-terminate complete | Delete infrastructure object | Deleting reason=WaitingForInfrastructureDeletion |
| Infra deleted | Bootstrap.ConfigRef exists | Infra gone | Delete bootstrap config | Deleting reason=WaitingForBootstrapDeletion |
| Bootstrap deleted | NodeRef set & allowed | Bootstrap gone | Delete Kubernetes node (with retry) | Deleting reason=DeletingNode |
| Node deleted | - | Node gone or not allowed | Remove finalizer | Object deleted by GC |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| InfrastructureProvisioned=true | InfraRef defined | Infra object not found | Set FailureReason, return error | FailureReason=InvalidConfiguration |
| NodeRef set | - | Node not found (not deleting) | Set FailureMessage | NodeHealthy=Unknown |
| Any | - | Drain timeout exceeded | Skip remaining drain | Continue to volume detach |
| Any | - | Volume detach timeout exceeded | Skip volume wait | Continue to pre-terminate |
| Any | - | Node deletion timeout exceeded | Skip node deletion | Remove finalizer anyway |
| Any | - | Generic API error | Requeue with error | Error logged, requeue |

## Status Fields

| Field | Description |
|-------|-------------|
| `status.phase` | Current phase: Pending, Provisioning, Running, Deleting, Failed |
| `status.nodeRef` | Reference to the associated Kubernetes Node |
| `status.addresses` | Machine addresses from infrastructure |
| `status.initialization.bootstrapDataSecretCreated` | Whether bootstrap data secret exists |
| `status.initialization.infrastructureProvisioned` | Whether infrastructure is provisioned |
| `status.certificatesExpiryDate` | Expiry date of machine certificates (control plane) |
| `status.deletion.nodeDrainStartTime` | When node drain started |
| `status.deletion.waitForNodeVolumeDetachStartTime` | When volume detach wait started |

## Conditions

### V1Beta2 Conditions

| Condition | Description |
|-----------|-------------|
| `BootstrapConfigReady` | Mirrors bootstrap config's Ready condition |
| `InfrastructureReady` | Mirrors infrastructure machine's Ready condition |
| `NodeReady` | Mirrors the Node's Ready condition |
| `NodeHealthy` | Overall node health based on multiple node conditions |
| `Deleting` | Set during deletion with progress details |
| `Updating` | Set during in-place updates (if feature enabled) |
| `UpToDate` | Whether machine matches desired spec |
| `Ready` | Summary of all conditions |
| `Available` | Whether machine is available for workloads |
| `Paused` | Set when machine or cluster is paused |

### V1Beta1 Conditions (Deprecated)

| Condition | Description |
|-----------|-------------|
| `Ready` | Summary condition with step counter during provisioning |
| `BootstrapReady` | Mirrors bootstrap config Ready state |
| `InfrastructureReady` | Mirrors infrastructure Ready state |
| `DrainingSucceeded` | Whether node drain completed successfully |
| `MachineHealthCheckSucceeded` | Set by MachineHealthCheck controller |
| `MachineOwnerRemediated` | Signals owner to remediate machine |

## Deletion Lifecycle Hooks

The Machine controller supports lifecycle hooks during deletion:

```mermaid
sequenceDiagram
    participant User
    participant MachineController
    participant HookOwner
    participant Node
    participant Infra
    
    User->>MachineController: Delete Machine
    MachineController->>MachineController: Check pre-drain hooks
    alt Has pre-drain hooks
        MachineController->>HookOwner: Wait for annotation removal
        HookOwner->>MachineController: Remove annotation
    end
    MachineController->>Node: Drain node
    MachineController->>Node: Wait for volume detach
    MachineController->>MachineController: Check pre-terminate hooks
    alt Has pre-terminate hooks
        MachineController->>HookOwner: Wait for annotation removal
        HookOwner->>MachineController: Remove annotation
    end
    MachineController->>Infra: Delete InfraMachine
    MachineController->>Node: Delete Node
    MachineController->>MachineController: Remove finalizer
```

### Hook Annotations

| Annotation Prefix | Phase | Description |
|------------------|-------|-------------|
| `pre-drain.delete.hook.machine.cluster.x-k8s.io/` | Before drain | Blocks node drain until removed |
| `pre-terminate.delete.hook.machine.cluster.x-k8s.io/` | Before infra delete | Blocks infrastructure deletion until removed |

## Watches

The Machine controller watches:

1. **Machine** - Primary resource
2. **Cluster** - For pause propagation and control plane initialization
3. **MachineSet** - For owner relationship
4. **MachineDeployment** - For owner relationship
5. **ClusterCache** - For workload cluster connection (node operations)

---

[← Back to Index](README.md) | [Previous: Cluster Controller](cluster_controller.md) | [Next: MachineSet Controller →](machineset_controller.md)
