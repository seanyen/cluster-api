# ClusterClass Reconciler - RefVersionsUpToDate Condition

**Reconciler**: `internal/controllers/clusterclass/clusterclass_controller.go`  
**Primary Reconciled Object**: `ClusterClass`  
**Condition Type Modeled**: `RefVersionsUpToDate`

## Overview

The `RefVersionsUpToDate` condition on a ClusterClass indicates whether all template references (`templateRefs`) in the ClusterClass are using the latest available API version. This condition helps operators identify when ClusterClass configurations need to be updated to use newer API versions of referenced templates.

---

## Condition Transition Table — `RefVersionsUpToDate` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `RefVersionsUpToDate`) | Next reconcile (Return) |
|---|---|---|---|
| `reconcileExternalReferencesError != nil` | No-op; internal error during reconciliation | `RefVersionsUpToDate=Unknown; Reason=InternalError; Message="Please check controller logs for errors"; ObservedGeneration=metadata.generation` | `none` |
| `len(outdatedExternalReferences) > 0` | No-op; report outdated refs | `RefVersionsUpToDate=False; Reason=RefVersionsNotUpToDate; Message="The following templateRefs are not using the latest apiVersion:\n* <Kind> <Name>: current: <version>, latest: <version>"; ObservedGeneration=metadata.generation` | `none` |
| `len(outdatedExternalReferences) == 0 && reconcileExternalReferencesError == nil` | No-op; all refs up to date | `RefVersionsUpToDate=True; Reason=RefVersionsUpToDate; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **External References Check**: The controller checks all template references in the ClusterClass (infrastructure templates, control plane templates, bootstrap templates, etc.) to determine if they are using the latest available API version.

2. **Informational Condition**: This condition is primarily informational and doesn't block ClusterClass functionality. Clusters can still be created using a ClusterClass with outdated references, but operators should update the references to use newer API versions when available.

3. **Message Format**: When outdated references are found, the message lists each outdated reference with:
   - The Kind of the referenced object
   - The Name of the referenced object  
   - The current API version being used
   - The latest available API version

4. **Watch-Driven**: The reconciler is watch-driven based on changes to the ClusterClass and its referenced template resources.

5. **Related Condition**: The `VariablesReady` condition (modeled separately) indicates whether variable discovery succeeded for the ClusterClass.
