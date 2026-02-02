# ExtensionConfig Reconciler - Discovered Condition

**Reconciler**: `internal/controllers/extensionconfig/extensionconfig_controller.go`  
**Primary Reconciled Object**: `ExtensionConfig`  
**Condition Type Modeled**: `Discovered`

## Overview

The `Discovered` condition on an ExtensionConfig indicates whether the RuntimeSDK has successfully discovered and registered the extension's handlers. This condition reflects the ability to communicate with the extension endpoint and discover its capabilities.

---

## Condition Transition Table — `Discovered` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Discovered`) | Next reconcile (Return) |
|---|---|---|---|
| `RuntimeClient.IsReady() == false` | No-op; registry not warmed up yet | (condition unchanged) | `requeueAfter: 10s` |
| `!deletionTimestamp.IsZero()` | Unregister ExtensionConfig from RuntimeClient registry | (condition unchanged; object being deleted) | `none` |
| `isPaused == true` (ReadOnly mode) | Validate ExtensionConfig; skip if paused | (condition unchanged) | `none` |
| `validateExtensionConfig(extensionConfig) returns error` (ReadOnly mode) | No-op; validation failed | `Discovered=False; Reason=NotDiscovered; Message="Error in discovery: <validation error>"; ObservedGeneration=metadata.generation` | `none` |
| `ReadOnly mode && validation succeeded` | Register ExtensionConfig in RuntimeClient registry | `Discovered=True; Reason=Discovered; Message=""; ObservedGeneration=metadata.generation` | `none` |
| `!ReadOnly mode && CABundle reconciliation error` | Attempt to reconcile CA bundle from Secret | (error returned; condition may not be updated) | `none` |
| `RuntimeClient.Discover() returns error` | Set handlers to empty; mark discovery failed | `Discovered=False; Reason=NotDiscovered; Message="Error in discovery: <error>"; ObservedGeneration=metadata.generation` | `none` |
| `RuntimeClient.Discover() succeeds` | Update status.handlers with discovered handlers; register ExtensionConfig | `Discovered=True; Reason=Discovered; Message=""; ObservedGeneration=metadata.generation` | `none` |

---

## Notes

1. **Registry Warmup**: The controller has a warmup phase during startup to ensure the RuntimeSDK registry is populated with existing ExtensionConfigs before other controllers start reconciling.

2. **ReadOnly vs Normal Mode**: 
   - **Normal Mode**: Reconciles CA bundle from Secrets, discovers extensions, and registers them
   - **ReadOnly Mode**: Only validates and registers already-configured ExtensionConfigs (used in secondary management clusters)

3. **CA Bundle Injection**: Similar to cert-manager's cainjector, the controller can inject CA bundles from Secrets into ExtensionConfigs using the `runtime.cluster.x-k8s.io/inject-ca-from-secret` annotation.

4. **Discovery Process**: The `RuntimeClient.Discover()` call:
   - Connects to the extension endpoint
   - Calls the Discovery endpoint to get supported hooks/handlers
   - Updates `status.handlers` with the discovered capabilities

5. **Registration**: After successful discovery (or validation in ReadOnly mode), the ExtensionConfig is registered in the RuntimeClient registry, making its handlers available to other controllers.

6. **Secret Watches**: The controller watches Secrets referenced in the `inject-ca-from-secret` annotation to update CA bundles when they change.

7. **Deletion Handling**: When an ExtensionConfig is deleted, it is unregistered from the RuntimeClient registry to prevent calls to the now-unavailable extension.
