# ClusterClass Reconciler - VariablesReady Condition

**Reconciler**: `internal/controllers/clusterclass/clusterclass_controller.go`  
**Primary Reconciled Object**: `ClusterClass`  
**Condition Type Modeled**: `VariablesReady`

## Overview

The `VariablesReady` condition on a ClusterClass indicates whether the variable definitions (both inline and from external RuntimeExtensions) have been successfully reconciled and are available for use by Clusters.

---

## Condition Transition Table — `VariablesReady` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `VariablesReady`) | Next reconcile (Return) |
|---|---|---|---|
| `variableDiscoveryError == nil && all inline variables valid && all external DiscoverVariables calls succeeded` | Update status.variables with all variable definitions | `VariablesReady=True; Reason=VariablesReady; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `error calling DiscoverVariables extension` | Aggregate error from extension calls | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="failed to call DiscoverVariables for patch <name>: <error>"; ObservedGeneration=metadata.generation` | `none` |
| `DiscoverVariables extension returned non-success status` | Aggregate error from extension response | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="patch <name> returned status <status> with message <msg>"; ObservedGeneration=metadata.generation` | `none` |
| `DiscoverVariables returned invalid variable definitions` | Validate variables and aggregate errors | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="<validation errors>"; ObservedGeneration=metadata.generation` | `none` |
| `error converting variables from v1beta1 to v1beta2` | Aggregate conversion errors | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="failed to convert variable <name> to v1beta2"; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Variable Sources**: ClusterClass variables can come from two sources:
   - **Inline**: Defined directly in `spec.variables[]`
   - **External**: Discovered via RuntimeExtension `DiscoverVariables` hooks from `spec.patches[].external.discoverVariablesExtension`

2. **Variable Validation**: All discovered variables are validated using `variables.ValidateClusterClassVariables()` before being added to status.

3. **Caching**: The controller uses a cache (`discoverVariablesCache`) to temporarily store DiscoverVariables responses to improve performance when multiple ClusterClasses use the same extension/settings.

4. **Variable Definitions in Status**: Successfully reconciled variables are stored in `status.variables[]` with:
   - `name`: Variable name
   - `definitions[]`: List of variable definitions with their source (inline or patch name)
   - `definitionsConflict`: True if multiple definitions exist with conflicting schemas

5. **RuntimeSDK Feature Gate**: External variable discovery only occurs when the `RuntimeSDK` feature gate is enabled.

6. **Watch-Driven**: The reconciler watches:
   - ClusterClass objects
   - ExtensionConfig objects (to re-reconcile when extensions change)
