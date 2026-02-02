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
| `error calling DiscoverVariables extension (RuntimeSDK enabled && patch.External.DiscoverVariablesExtension != "")` | Aggregate error; return error to trigger retry | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="VariableDiscovery failed: failed to call DiscoverVariables for patch <name>: <error>"; ObservedGeneration=metadata.generation` | `requeue` (error return) |
| `DiscoverVariables extension returned non-success status` | Aggregate error; return error to trigger retry | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="VariableDiscovery failed: patch <name> returned status <status> with message <msg>"; ObservedGeneration=metadata.generation` | `requeue` (error return) |
| `error converting variables from v1beta1 to v1beta2` | Aggregate conversion error; return error to trigger retry | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="VariableDiscovery failed: failed to convert variable <name> to v1beta2"; ObservedGeneration=metadata.generation` | `requeue` (error return) |
| `DiscoverVariables returned variables that fail ValidateClusterClassVariables()` | Aggregate validation errors; return error to trigger retry | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="VariableDiscovery failed: <validation errors>"; ObservedGeneration=metadata.generation` | `requeue` (error return) |
| `status.variables contains variables with definitionsConflict=true` | Set variableDiscoveryError; return error to trigger retry | `VariablesReady=False; Reason=VariableDiscoveryFailed; Message="VariableDiscovery failed: the following variables have conflicting schemas: <names>"; ObservedGeneration=metadata.generation` | `requeue` (error return) |
| `all inline variables processed && all external DiscoverVariables calls succeeded && no conflicting definitions` | Update status.variables with sorted variable definitions | `VariablesReady=True; Reason=VariablesReady; Message=""; ObservedGeneration=metadata.generation` | `none` |

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

7. **Error Handling**: All error paths in `reconcileVariables` return an error to the controller runtime, which triggers a requeue with exponential backoff. The `variableDiscoveryError` is captured in scope for status update before the error is returned.

8. **Stable Ordering**: Variables in `status.variables` are alphabetically sorted by name to prevent unnecessary status updates.
