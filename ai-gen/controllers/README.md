# Controllers Package Documentation

## Overview

The `controllers` package in Cluster API serves as the main entry point for controller reconcilers and shared controller utilities. This package provides:

1. **Reconciler Aliases** - Public wrappers around internal controller implementations
2. **ClusterCache** - Connection management for workload clusters
3. **CRDMigrator** - CRD storage version migration utilities
4. **External** - Utilities for working with external/provider objects
5. **Remote** - Client creation for remote workload clusters
6. **NodeRefUtil** - Node state evaluation utilities

## Package Structure

```mermaid
flowchart TB
    subgraph controllers["controllers/"]
        alias[alias.go<br/>Reconciler Aliases]
        doc[doc.go]
        
        subgraph clustercache["clustercache/"]
            cc[cluster_cache.go]
            ca[cluster_accessor.go]
            cac[cluster_accessor_client.go]
            metrics[metrics.go]
        end
        
        subgraph crdmigrator["crdmigrator/"]
            crm[crd_migrator.go]
        end
        
        subgraph external["external/"]
            tracker[tracker.go]
            util[util.go]
            types[types.go]
        end
        
        subgraph remote["remote/"]
            cluster[cluster.go]
            restconfig[restconfig.go]
        end
        
        subgraph noderefutil["noderefutil/"]
            nodeutil[util.go]
        end
    end
    
    style controllers fill:#e1f5fe,stroke:#333
    style clustercache fill:#c8e6c9,stroke:#333
    style crdmigrator fill:#fff3e0,stroke:#333
    style external fill:#f3e5f5,stroke:#333
    style remote fill:#fce4ec,stroke:#333
    style noderefutil fill:#e8f5e9,stroke:#333
```

## Reconciler Aliases

The `alias.go` file provides public types that wrap internal reconciler implementations:

| Reconciler | Resource | Description |
|------------|----------|-------------|
| `ClusterReconciler` | Cluster | Reconciles Cluster objects, manages remote connection grace period |
| `MachineReconciler` | Machine | Reconciles Machine objects, syncs labels/annotations, uses RuntimeClient |
| `MachineSetReconciler` | MachineSet | Reconciles MachineSet objects, supports preflight checks |
| `MachineDeploymentReconciler` | MachineDeployment | Reconciles MachineDeployment objects |
| `MachineHealthCheckReconciler` | MachineHealthCheck | Reconciles MachineHealthCheck objects |
| `ClusterTopologyReconciler` | Cluster (topology) | Reconciles managed topology for Clusters |
| `MachineDeploymentTopologyReconciler` | MachineDeployment | Handles topology-owned MachineDeployment deletion and template cleanup |
| `MachineSetTopologyReconciler` | MachineSet | Handles topology-owned MachineSet deletion and template cleanup |
| `ClusterClassReconciler` | ClusterClass | Reconciles ClusterClass objects, exposes `Reconcile()` for testing |
| `ClusterResourceSetReconciler` | ClusterResourceSet | Reconciles ClusterResourceSet objects, requires `partialSecretCache` |
| `ClusterResourceSetBindingReconciler` | ClusterResourceSetBinding | Reconciles ClusterResourceSetBinding objects |
| `MachinePoolReconciler` | MachinePool | Reconciles MachinePool objects |
| `ExtensionConfigReconciler` | ExtensionConfig | Reconciles ExtensionConfig objects, supports read-only mode |

### Common Reconciler Fields

Most reconcilers share these common fields (not all reconcilers have all fields):

```go
// Common fields found across multiple reconcilers:
type CommonReconcilerFields struct {
    // Client is a cached client for reading/writing resources
    Client client.Client
    
    // APIReader is a live (non-cached) client for critical reads
    // (avoids race conditions from stale cache)
    APIReader client.Reader
    
    // ClusterCache provides cached connections to workload clusters
    // Not all reconcilers need workload cluster access
    ClusterCache clustercache.ClusterCache
    
    // RuntimeClient is used for calling runtime extensions
    // Only present on reconcilers that use runtime hooks
    RuntimeClient runtimeclient.Client
    
    // WatchFilterValue filters resources by label before reconciliation
    WatchFilterValue string
}
```

## Sub-packages

### [ClusterCache](clustercache-controller.md)

Manages cached connections to workload clusters including:
- Cached and uncached clients
- REST configurations
- Health probing
- Watch management

### [CRDMigrator](crdmigrator-controller.md)

Handles CRD migrations during upgrades:
- Storage version migration
- ManagedFields cleanup

### [External](external-package.md)

Utilities for working with external/provider objects:
- Dynamic object tracking
- Template cloning
- Object status helpers

### [Remote](remote-package.md)

Client creation for remote workload clusters:
- REST configuration
- Client creation
- User-Agent management

### [NodeRefUtil](noderefutil-package.md)

Node state evaluation utilities:
- Node readiness checks
- Node availability checks
- Node reachability checks

## Component Interaction

```mermaid
flowchart TB
    subgraph Reconcilers["Public Reconcilers (alias.go)"]
        CR[ClusterReconciler]
        MR[MachineReconciler]
        MSR[MachineSetReconciler]
        MDR[MachineDeploymentReconciler]
    end
    
    subgraph Internal["Internal Controllers"]
        IC[internal/controllers/cluster]
        IM[internal/controllers/machine]
        IMS[internal/controllers/machineset]
        IMD[internal/controllers/machinedeployment]
    end
    
    subgraph Utilities["Shared Utilities"]
        CC[ClusterCache]
        EXT[External]
        REM[Remote]
        NRU[NodeRefUtil]
    end
    
    CR -->|delegates to| IC
    MR -->|delegates to| IM
    MSR -->|delegates to| IMS
    MDR -->|delegates to| IMD
    
    IC -->|uses| CC
    IC -->|uses| EXT
    
    IM -->|uses| CC
    IM -->|uses| EXT
    IM -->|uses| NRU
    
    IMS -->|uses| CC
    IMD -->|uses| CC
    
    CC -->|uses| REM
    
    style Reconcilers fill:#4a9eff,stroke:#333,color:#fff
    style Utilities fill:#90EE90,stroke:#333
```

## Setup Pattern

All reconcilers follow a consistent setup pattern:

```go
func (r *ReconcilerType) SetupWithManager(ctx context.Context, mgr ctrl.Manager, options controller.Options) error {
    return (&internalReconciler{
        Client:           r.Client,
        APIReader:        r.APIReader,
        ClusterCache:     r.ClusterCache,
        WatchFilterValue: r.WatchFilterValue,
        // ... other fields
    }).SetupWithManager(ctx, mgr, options)
}
```

## Usage Example

```go
package main

import (
    "context"
    
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    
    "sigs.k8s.io/cluster-api/controllers"
    "sigs.k8s.io/cluster-api/controllers/clustercache"
    "sigs.k8s.io/cluster-api/controllers/remote"
)

func main() {
    ctx := context.Background()
    mgr, _ := ctrl.NewManager(cfg, ctrl.Options{})
    
    // Setup ClusterCache (returns ClusterCache interface)
    clusterCache, _ := clustercache.SetupWithManager(ctx, mgr, clustercache.Options{
        SecretClient:     secretClient,
        WatchFilterValue: "",
        Client: clustercache.ClientOptions{
            Timeout:   10 * time.Second,
            QPS:       20,
            Burst:     30,
            UserAgent: remote.DefaultClusterAPIUserAgent("cluster-api-controller"),
        },
    }, controller.Options{})
    
    // Setup Cluster Reconciler
    clusterReconciler := &controllers.ClusterReconciler{
        Client:                      mgr.GetClient(),
        APIReader:                   mgr.GetAPIReader(),
        ClusterCache:                clusterCache,
        RemoteConnectionGracePeriod: 10 * time.Minute,
    }
    _ = clusterReconciler.SetupWithManager(ctx, mgr, controller.Options{})
    
    // Setup Machine Reconciler (requires RuntimeClient)
    machineReconciler := &controllers.MachineReconciler{
        Client:        mgr.GetClient(),
        APIReader:     mgr.GetAPIReader(),
        ClusterCache:  clusterCache,
        RuntimeClient: runtimeClient, // from exp/runtime/client
    }
    _ = machineReconciler.SetupWithManager(ctx, mgr, controller.Options{})
    
    // ... setup other reconcilers
    
    mgr.Start(ctx)
}
```

## Documentation Index

| Document | Description | Key KRTT Tables |
|----------|-------------|-----------------|
| [clustercache-controller.md](clustercache-controller.md) | ClusterCache controller and workload cluster connection management | Reconcile, Connect/Disconnect, Health Checking |
| [crdmigrator-controller.md](crdmigrator-controller.md) | CRD migration controller for storage version and managedFields | Main Reconcile, Storage Version Migration, ManagedFields Cleanup |
| [external-package.md](external-package.md) | External object utilities for provider resources | ObjectTracker Watch, External Object Operations |
| [remote-package.md](remote-package.md) | Remote cluster client creation utilities | NewClusterClient, RESTConfig |
| [noderefutil-package.md](noderefutil-package.md) | Node state evaluation utilities | Node State Evaluation, Node Availability |

## KRTT Summary

Each sub-package documentation contains detailed Kubernetes Reconciler Transition Tables (KRTT) that document:

1. **Observed Status** - Current state of the resource
2. **Desired Spec** - Intended state from the specification
3. **Trigger / Condition** - What triggers the reconciliation
4. **Reconciliation Action** - Idempotent action taken
5. **Resulting Status** - Updated status after action

See individual package documentation for complete KRTT tables.
