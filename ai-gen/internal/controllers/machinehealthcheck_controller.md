# MachineHealthCheck Controller

The MachineHealthCheck Controller monitors the health of machines and triggers remediation when machines become unhealthy based on configurable health checks.

## Overview

```mermaid
flowchart TB
    subgraph "MachineHealthCheck Controller"
        R[Reconcile] --> F{Fetch MHC}
        F -->|Not Found| End[Return]
        F -->|Found| GC[Get Cluster]
        GC --> P{Paused?}
        P -->|Yes| PC[Set Paused Condition & Return]
        P -->|No| RN[reconcile]
    end
    
    subgraph "Reconcile Flow"
        SO[Set Owner Reference]
        GT[Get Targets]
        HC[Health Check Targets]
        CR{Check Remediation Allowed}
        REM[Remediate Unhealthy]
    end
    
    RN --> SO --> GT --> HC --> CR
    CR -->|Allowed| REM
    CR -->|Not Allowed| SC[Short Circuit]
```

## Health Checking Flow

```mermaid
flowchart TD
    A[Get Target Machines] --> B[For Each Machine]
    B --> C{Has NodeRef?}
    C -->|No| D{Timeout Exceeded?}
    D -->|Yes| E[Mark Unhealthy: NodeStartupTimeout]
    D -->|No| F[Mark Healthy: Waiting for Node]
    C -->|Yes| G[Get Node from Cluster]
    G -->|Not Found| H[Mark Unhealthy: NodeNotFound]
    G -->|Found| I[Check Node Conditions]
    I --> J{Conditions Healthy?}
    J -->|Yes| K[Mark Healthy]
    J -->|No| L{Timeout Exceeded?}
    L -->|Yes| M[Mark Unhealthy: UnhealthyCondition]
    L -->|No| N[Mark Pending: Waiting]
```

## Remediation Logic

```mermaid
flowchart TD
    A[Get Unhealthy Targets] --> B[Calculate MaxUnhealthy]
    B --> C{Unhealthy Count > MaxUnhealthy?}
    C -->|Yes| D[Short Circuit Remediation]
    D --> E[Set RemediationAllowed=False]
    C -->|No| F[Set RemediationAllowed=True]
    F --> G[For Each Unhealthy Target]
    G --> H{External Remediation?}
    H -->|Yes| I[Set MachineOwnerRemediated Condition]
    H -->|No| J{Has Owner?}
    J -->|Yes| I
    J -->|No| K[Delete Machine Directly]
```

## KRTT - Kubernetes Reconciler Transition Table

### Health Checking

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Machine without NodeRef | - | NodeStartupTimeout not exceeded | Wait for node association | HealthCheckSucceeded=Unknown |
| Machine without NodeRef | - | NodeStartupTimeout exceeded | Mark machine unhealthy | HealthCheckSucceeded=False |
| Machine with NodeRef | - | Node not found | Mark machine unhealthy | HealthCheckSucceeded=False |
| Machine with NodeRef | - | Node Ready=True | Mark machine healthy | HealthCheckSucceeded=True |
| Machine with NodeRef | - | Node Ready=False, timeout not exceeded | Wait for node recovery | HealthCheckSucceeded=Unknown |
| Machine with NodeRef | - | Node Ready=False, timeout exceeded | Mark machine unhealthy | HealthCheckSucceeded=False |

### Remediation Decision

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Unhealthy count ≤ maxUnhealthy | unhealthyLessThanOrEqualTo set | Within threshold | Allow remediation | RemediationAllowed=True |
| Unhealthy count > maxUnhealthy | unhealthyLessThanOrEqualTo set | Over threshold | Short circuit | RemediationAllowed=False |
| Unhealthy count in range | unhealthyInRange set | Within range | Allow remediation | RemediationAllowed=True |
| Unhealthy count not in range | unhealthyInRange set | Outside range | Short circuit | RemediationAllowed=False |

### Remediation Execution

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Machine unhealthy | Remediation allowed | No owner controller | Delete machine directly | Machine deleted |
| Machine unhealthy | Remediation allowed | Has MachineSet owner | Set MachineOwnerRemediated condition | Owner handles deletion |
| Machine unhealthy | Remediation allowed | Has ControlPlane owner | Set MachineOwnerRemediated condition | Owner handles deletion |
| Machine unhealthy | External remediation | ExternalRemediationTemplate set | Create remediation request | External controller handles |

### Error Handling

| Observed Status | Desired Spec | Trigger / Condition | Reconciliation Action | Resulting Status |
|:---|:---|:---|:---|:---|
| Can't get cluster | - | Cluster not found | Log error, requeue | Error logged |
| Can't list machines | - | API error | Log error, requeue | Error logged |
| Can't get node | - | Remote cluster error | Mark node as unavailable | NodeHealthy=Unknown |

## Health Check Configuration

```yaml
apiVersion: cluster.x-k8s.io/v1beta2
kind: MachineHealthCheck
metadata:
  name: my-mhc
  namespace: default
spec:
  clusterName: my-cluster
  selector:
    matchLabels:
      cluster.x-k8s.io/deployment-name: my-deployment
  
  # Node startup timeout (v1beta2 structure)
  nodeStartupTimeout: 5m
  
  # Unhealthy conditions to check
  unhealthyConditions:
    - type: Ready
      status: "False"
      timeout: 5m
    - type: Ready
      status: "Unknown"
      timeout: 5m
  
  # Remediation threshold using triggerIf
  triggerIf:
    unhealthyLessThanOrEqualTo: "100%"  # Short-circuit threshold
    # OR use unhealthyInRange for range-based threshold
    # unhealthyInRange: "0-40%"
  
  # Optional: External remediation template
  remediationTemplate:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: MyRemediationTemplate
    name: my-remediation-template
```

## MaxUnhealthy Calculation

```mermaid
flowchart TD
    A[Get Total Targets] --> B{MaxUnhealthy Type}
    B -->|Integer| C[Use Direct Value]
    B -->|Percentage| D[Calculate: Total * Percentage]
    C --> E[Compare Unhealthy Count]
    D --> E
    E --> F{Unhealthy > MaxUnhealthy?}
    F -->|Yes| G[Short Circuit]
    F -->|No| H[Allow Remediation]
```

## External Remediation

When `spec.remediationTemplate` is set, the controller creates external remediation requests:

```mermaid
sequenceDiagram
    participant MHC as MachineHealthCheck
    participant Machine
    participant ExtRem as External Remediation
    participant Controller as External Controller
    
    MHC->>Machine: Detect Unhealthy
    MHC->>ExtRem: Create Remediation Request
    ExtRem->>Controller: Trigger
    Controller->>Machine: Custom Remediation
    Controller->>ExtRem: Update Status
    MHC->>MHC: Check Remediation Result
```

## Status Fields

| Field | Description |
|-------|-------------|
| `status.expectedMachines` | Total machines matching selector |
| `status.currentHealthy` | Number of healthy machines |
| `status.remediationsAllowed` | Number of remediations currently allowed |
| `status.targets` | List of machine names being monitored |

## Conditions

| Condition | Description |
|-----------|-------------|
| `RemediationAllowed` | Whether remediation is currently permitted |

### Machine Conditions Set by MHC

| Condition | Description |
|-----------|-------------|
| `HealthCheckSucceeded` | Whether health check passed |
| `MachineOwnerRemediated` | Signals owner to remediate machine |

## Target Selection

Machines are selected using a label selector:

```yaml
spec:
  selector:
    matchLabels:
      cluster.x-k8s.io/deployment-name: my-md
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
```

## Unhealthy Conditions

The MHC checks multiple node conditions:

```mermaid
flowchart TD
    A[Node] --> B[Check Ready Condition]
    A --> C[Check MemoryPressure]
    A --> D[Check DiskPressure]
    A --> E[Check PIDPressure]
    B --> F{Any Unhealthy?}
    C --> F
    D --> F
    E --> F
    F -->|Yes| G[Start Timeout]
    G --> H{Timeout Exceeded?}
    H -->|Yes| I[Mark Unhealthy]
    H -->|No| J[Continue Monitoring]
```

## Watches

The MachineHealthCheck controller watches:

1. **MachineHealthCheck** - Primary resource
2. **Machine** - Machines matching the selector
3. **Cluster** - For pause propagation
4. **Node** (remote) - For health status

---

[← Back to Index](README.md) | [Previous: MachinePool Controller](machinepool_controller.md) | [Next: ClusterClass Controller →](clusterclass_controller.md)
