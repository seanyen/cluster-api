# Cluster Controller

The Cluster Controller manages the lifecycle of `Cluster` resources, coordinating infrastructure provisioning, control plane initialization, and cluster-wide operations.

## Overview

```mermaid
flowchart TB
    subgraph "Cluster Controller"
        R[Reconcile] --> F{Fetch Cluster}
        F -->|Not Found| End[Return]
        F -->|Found| Fin[Add Finalizer]
        Fin --> P{Paused?}
        P -->|Yes| PC[Set Paused Condition & Return]
        P -->|No| CLS{ClusterClass Defined?}
        CLS -->|Yes & Refs Missing| WT[Wait for Topology Generation]
        CLS -->|No or Refs Ready| D{Deleting?}
        D -->|Yes| RD[reconcileDelete]
        D -->|No| RN[reconcileNormal]
    end
    
    subgraph "Always Reconcile Phases"
        I[reconcileInfrastructure]
        CP[reconcileControlPlane]
        GD[getDescendants]
    end
    
    subgraph "Normal Reconcile"
        K[reconcileKubeconfig]
        CPI[reconcileV1Beta1ControlPlaneInitialized]
    end
    
    subgraph "Delete Reconcile"
        HOOK{BeforeClusterDelete Hook?}
        DC[Delete Children]
        DCP[Delete Control Plane]
        DI[Delete Infrastructure]
        RF[Remove Finalizer]
    end
    
    RN --> I --> CP --> GD --> K --> CPI
    RD --> I --> CP --> GD --> HOOK --> DC --> DCP --> DI --> RF
```

## Reconciliation Phases

### 1. reconcileInfrastructure

Reconciles the `spec.infrastructureRef` object.

```mermaid
flowchart TD
    A[Start] --> B{InfraRef Defined?}
    B -->|No| C[Mark InfrastructureProvisioned=true]
    B -->|Yes| D[Get External Object]
    D -->|Not Found| E{Deleting?}
    E -->|Yes| F[Tolerate - Return]
    E -->|No| G{Was Provisioned?}
    G -->|Yes| H[Error: Deleted after provisioned]
    G -->|No| I[Requeue after 30s]
    D -->|Found| J[Set Owner Ref & Labels]
    J --> K{Provisioned?}
    K -->|No| L[Log & Wait]
    K -->|Yes| M[Get ControlPlaneEndpoint]
    M --> N[Get FailureDomains]
    N --> O[Set InfrastructureProvisioned=true]
```

### 2. reconcileControlPlane

Reconciles the `spec.controlPlaneRef` object.

```mermaid
flowchart TD
    A[Start] --> B{ControlPlaneRef Defined?}
    B -->|No| End[Return]
    B -->|Yes| C[Get External Object]
    C -->|Not Found| D{Deleting?}
    D -->|Yes| E[Tolerate - Return]
    D -->|No| F{Was Initialized?}
    F -->|Yes| G[Error: Deleted after initialized]
    F -->|No| H[Requeue after 30s]
    C -->|Found| I[Set Owner Ref & Labels]
    I --> J{Initialized?}
    J -->|No| K[Log & Wait]
    J -->|Yes| L[Get ControlPlaneEndpoint]
    L --> M[Set ControlPlaneInitialized=true]
```

### 3. reconcileDelete

Handles cluster deletion with proper ordering. The controller follows a strict deletion sequence:
1. Wait for BeforeClusterDelete hook (if RuntimeSDK + ClusterTopology enabled)
2. Delete all descendant resources (MachineDeployments, MachineSets, Machines, MachinePools)
3. Delete ControlPlane object
4. Delete Infrastructure object
5. Remove finalizer

```mermaid
flowchart TD
    A[Start] --> B{RuntimeSDK + ClusterTopology?}
    B -->|Yes| C{Cluster has managed topology?}
    C -->|Yes| D{ok-to-delete annotation?}
    D -->|No| E[Set Deleting: WaitingForBeforeDeleteHook]
    D -->|Yes| F[Continue]
    C -->|No| F
    B -->|No| F
    F --> G{getDescendants succeeded?}
    G -->|No| H[Set Deleting: InternalError]
    G -->|Yes| I{Has descendant children?}
    I -->|Yes| J[Delete owned children]
    J --> K{Descendants pending delete?}
    K -->|Yes| L[Set Deleting: WaitingForWorkersDeletion]
    K -->|No| M{ControlPlaneRef defined?}
    I -->|No| M
    M -->|Yes & CP exists| N[Delete ControlPlane]
    N --> O[Set Deleting: WaitingForControlPlaneDeletion]
    M -->|No or CP deleted| P{InfrastructureRef defined?}
    P -->|Yes & Infra exists| Q[Delete Infrastructure]
    Q --> R[Set Deleting: WaitingForInfrastructureDeletion]
    P -->|No or Infra deleted| S[Set Deleting: DeletionCompleted]
    S --> T[Remove Finalizer]
```

## KRTT - Kubernetes Reconciler Transition Table

### Normal Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Phase=Pending, No InfraRef | InfraRef not defined | Initial creation | Skip infra reconciliation, mark InfrastructureProvisioned=true | Phase=Provisioning |
| Phase=Pending | InfraRef defined | Initial creation | Get/create infrastructure object, set owner reference | Phase=Provisioning, InfrastructureReady=False |
| InfrastructureReady=False | InfraRef defined | Infra object created | Watch infra object, wait for provisioned=true | No change, requeue |
| InfrastructureReady=False | InfraRef defined | Infra reports provisioned=true | Copy ControlPlaneEndpoint, FailureDomains to Cluster | InfrastructureProvisioned=True |
| ControlPlaneReady=False | ControlPlaneRef defined | Infra ready | Get/create control plane object, set owner reference | ControlPlaneReady condition mirrors CP |
| ControlPlaneReady=False | ControlPlaneRef defined | CP reports initialized=true | Copy ControlPlaneEndpoint to Cluster | ControlPlaneInitialized=True |
| ControlPlaneReady=True | Valid endpoint | CP initialized, no Kubeconfig | Generate Kubeconfig secret (if no CP ref) | Kubeconfig secret created |
| Phase=Provisioning | All refs ready | Infra provisioned + endpoint valid | Update phase | Phase=Provisioned |

### Deletion Reconciliation

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| DeletionTimestamp!=nil | - | User deletes Cluster | Check BeforeClusterDelete hook (if RuntimeSDK) | Deleting condition set |
| Has descendants | - | Children exist | Delete owned MachineDeployments, MachineSets, Machines | Requeue after 5s |
| No worker descendants | ControlPlaneRef exists | CP still present | Delete ControlPlane object | Wait for CP deletion |
| No CP | InfraRef exists | Infra still present | Delete Infrastructure object | Wait for Infra deletion |
| No CP, No Infra | - | All children gone | Remove finalizer | Object deleted by GC |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| InfrastructureProvisioned=true | InfraRef defined | Infra object not found | Return error - infra deleted after provisioned | Error logged |
| ControlPlaneInitialized=true | CPRef defined | CP object not found | Return error - CP deleted after initialized | Error logged |
| Any | - | Generic API error | Requeue with error | Error logged, requeue |
| Any | - | Object paused | Set Paused condition, return | Paused=True |

## Status Fields

The Cluster controller manages the following status fields:

| Field | Description |
|-------|-------------|
| `status.phase` | Current phase: Pending, Provisioning, Provisioned, Deleting, Failed |
| `status.initialization.infrastructureProvisioned` | Whether infrastructure is provisioned |
| `status.initialization.controlPlaneInitialized` | Whether control plane is initialized |
| `status.failureDomains` | Available failure domains from infrastructure |
| `status.controlPlane.replicas` | Current number of control plane replicas |
| `status.controlPlane.readyReplicas` | Number of ready control plane replicas |
| `status.controlPlane.availableReplicas` | Number of available control plane replicas |
| `status.controlPlane.upToDateReplicas` | Number of up-to-date control plane replicas |
| `status.controlPlane.desiredReplicas` | Desired number of control plane replicas |
| `status.workers.replicas` | Total worker machine replicas |
| `status.workers.readyReplicas` | Number of ready worker replicas |
| `status.workers.availableReplicas` | Number of available worker replicas |
| `status.workers.upToDateReplicas` | Number of up-to-date worker replicas |
| `status.workers.desiredReplicas` | Desired number of worker replicas |

## Conditions

### V1Beta2 Conditions

| Condition | Description |
|-----------|-------------|
| `InfrastructureReady` | Infrastructure cluster is provisioned and ready |
| `ControlPlaneAvailable` | Control plane is available |
| `ControlPlaneInitialized` | First control plane machine has NodeRef |
| `ControlPlaneMachinesReady` | All control plane machines are ready |
| `ControlPlaneMachinesUpToDate` | All control plane machines match desired spec |
| `WorkersAvailable` | Worker resources are available |
| `WorkerMachinesReady` | All worker machines are ready |
| `WorkerMachinesUpToDate` | All worker machines match desired spec |
| `RemoteConnectionProbe` | Health of connection to workload cluster |
| `RollingOut` | A rollout is in progress |
| `ScalingUp` | Cluster is scaling up |
| `ScalingDown` | Cluster is scaling down |
| `Remediating` | Unhealthy machines are being remediated |
| `Deleting` | Set during deletion with progress details |
| `Available` | Overall cluster availability (summary) |
| `Paused` | Set when cluster is paused |

### V1Beta1 Conditions (Deprecated)

| Condition | Description |
|-----------|-------------|
| `Ready` | Summary condition of ControlPlaneReady + InfrastructureReady |
| `ControlPlaneReady` | Mirrors control plane object's ready condition |
| `InfrastructureReady` | Mirrors infrastructure object's ready condition |
| `ControlPlaneInitialized` | Control plane is initialized |

## Watches

The Cluster controller watches:

1. **Cluster** - Primary resource
2. **Machine** - Control plane machines for initialization check
3. **MachineDeployment** - For status aggregation
4. **MachinePool** - For status aggregation (if feature enabled)
5. **ClusterCache** - For remote connection health

---

[← Back to Index](README.md) | [Next: Machine Controller →](machine_controller.md)
