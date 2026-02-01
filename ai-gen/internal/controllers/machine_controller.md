# Machine Controller

The Machine Controller manages the lifecycle of individual `Machine` resources, coordinating bootstrap configuration, infrastructure provisioning, node association, and machine deletion with proper cleanup.

## Overview

```mermaid
flowchart TB
    subgraph "Machine Controller"
        R[Reconcile] --> F{Fetch Machine}
        F -->|Not Found| End[Return]
        F -->|Found| Fin[Add Finalizer]
        Fin --> GC[Get Cluster]
        GC --> P{Paused?}
        P -->|Yes| PC[Set Paused Condition]
        P -->|No| D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph "Always Reconcile Phases"
        OL[reconcileMachineOwnerAndLabels]
        B[reconcileBootstrap]
        I[reconcileInfrastructure]
        N[reconcileNode]
        CE[reconcileCertificateExpiry]
    end
    
    subgraph "Normal Only"
        IPU[reconcileInPlaceUpdate]
    end
    
    subgraph "Delete Phases"
        PreDrain[Pre-Drain Hook]
        Drain[Drain Node]
        VolumeDetach[Wait Volume Detach]
        PreTerm[Pre-Terminate Hook]
        DelInfra[Delete Infrastructure]
        DelBoot[Delete Bootstrap]
        DelNode[Delete Node]
        RemFin[Remove Finalizer]
    end
    
    RN --> OL --> B --> I --> N --> CE --> IPU
    RD --> OL --> B --> I --> N --> CE --> PreDrain --> Drain --> VolumeDetach --> PreTerm --> DelInfra --> DelBoot --> DelNode --> RemFin
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
    A[Start] --> B{Delete Node Allowed?}
    B -->|No| Skip[Skip to Infrastructure]
    B -->|Yes| C{Pre-Drain Hooks?}
    C -->|Yes| D[Wait for hooks]
    C -->|No| E{Drain Allowed?}
    E -->|No| F[Skip drain]
    E -->|Yes| G[Drain Node]
    G -->|In Progress| H[Requeue]
    G -->|Complete| I{Volume Detach Allowed?}
    I -->|No| J[Skip volume detach]
    I -->|Yes| K[Wait for Volumes]
    K -->|In Progress| L[Requeue]
    K -->|Complete| M{Pre-Terminate Hooks?}
    M -->|Yes| N[Wait for hooks]
    M -->|No| Skip
    Skip --> O[Delete Infrastructure]
    O -->|In Progress| P[Wait]
    O -->|Complete| Q[Delete Bootstrap]
    Q -->|In Progress| R[Wait]
    Q -->|Complete| S[Delete Node]
    S --> T[Remove Finalizer]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Phase=Pending | Bootstrap.ConfigRef defined | Initial creation | Get bootstrap object, set owner ref | Phase=Provisioning |
| BootstrapReady=False | ConfigRef defined | Bootstrap created | Watch bootstrap, wait for dataSecretCreated | BootstrapReady mirrors bootstrap |
| BootstrapReady=False | ConfigRef defined | dataSecretCreated=true | Copy dataSecretName to Machine.spec | BootstrapDataSecretCreated=True |
| InfrastructureReady=False | InfraRef defined | Bootstrap ready | Get infra object, set owner ref | InfrastructureReady mirrors infra |
| InfrastructureReady=False | InfraRef defined | Infra provisioned=true | Wait for ProviderID | InfrastructureReady=False |
| InfrastructureReady=False | InfraRef defined | ProviderID set | Copy ProviderID, addresses, failureDomain | InfrastructureProvisioned=True |
| NodeRef=nil | ProviderID set | Infra ready | Find node by ProviderID | NodeRef set if found |
| NodeRef set | - | Node exists | Sync labels/annotations to node | NodeHealthy/Ready updated |
| Phase=Provisioning | All ready | Node associated | Update phase | Phase=Running |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes Machine | Check delete allowed | Deleting condition set |
| Has pre-drain hook annotation | - | Hook annotation present | Wait for hook owner to remove annotation | PreDrainDeleteHookSucceeded=False |
| No pre-drain hooks | NodeRef set | Pre-drain complete | Start node drain | DrainingSucceeded=False |
| Draining in progress | - | Pods being evicted | Continue draining, handle timeouts | Deleting reason=DrainingNode |
| Draining complete | - | All pods evicted/skipped | Start volume detach wait | DrainingSucceeded=True |
| Waiting for volumes | - | Volumes attached | Wait for volume detachment | Deleting reason=WaitingForVolumeDetach |
| Volumes detached | - | All volumes detached | Check pre-terminate hooks | VolumeDetachSucceeded=True |
| Has pre-terminate hook annotation | - | Hook annotation present | Wait for hook owner to remove annotation | PreTerminateDeleteHookSucceeded=False |
| No pre-terminate hooks | InfraRef exists | Pre-terminate complete | Delete infrastructure object | Deleting reason=WaitingForInfrastructureDeletion |
| Infra deleted | Bootstrap.ConfigRef exists | Infra gone | Delete bootstrap object | Deleting reason=WaitingForBootstrapDeletion |
| Bootstrap deleted | NodeRef set | Bootstrap gone | Delete Kubernetes node | Deleting reason=DeletingNode |
| Node deleted | - | Node gone | Remove finalizer | Object deleted by GC |

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
