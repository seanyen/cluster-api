## LLM-agent instructions (read carefully)

### Output and workspace rules

1. **Only produce outcome tables in Markdown.**  
   - Your final deliverable MUST be Markdown tables (like the template below).
   - Do not output code, YAML manifests, prose-only writeups, or mixed formats as the final deliverable.

2. **Write tables ONLY under `modeling/` in the workspace.**  
   - Create or update Markdown files under the `modeling/` folder (e.g., `modeling/<reconciler-name>-ready.md`).
   - Do NOT write tables outside `modeling/`.
   - Do NOT create or modify non-Markdown artifacts for this task.

3. **If modeling tables already exist, prioritize NEW reconcilers first.**  
   - First, identify reconcilers that **do not yet have a corresponding table** under `modeling/` and generate tables for them.
   - Only after all discovered reconcilers have tables should you review or refine existing tables.

### Reconcilers discovery rules

4. **Read these folders in the workspace to find reconcilers (scan for controller/reconciler code):**
   - `internal/controllers`
   - `controllers`
   - `bootstrap/kubeadm/controllers`
   - `controlplane/kubeadm/controllers`

5. **For each reconciler you model:**
   - Identify the primary reconciled object and its dependent resources.
   - Derive transitions from the reconciler’s control flow and observed states (reads), not from guesses.

---

## Modeling rules (single-condition table)

6. **One table = one Condition type.**  
   - Pick exactly one condition type name (example: `Ready`) and use it for **every** row in the table.  
   - **Never** update or describe transitions for additional condition types (e.g., `Reconciling`, `Stalled`, `Degraded`) in the same table.

7. **Each row represents one transition rule** for that single condition type:
   - **Guard**: a side-effect-free predicate over observed inputs (`spec`, `metadata`, `status`, and live reads).
   - **Ensure**: idempotent reconcile action(s) taken when Guard is true.
   - **Condition transition**: how the **single condition type** changes (`Status`, `Reason`, `Message`, `ObservedGeneration`).
   - **Next reconcile**: what the reconciler returns (`requeue`, `requeueAfter`, or `none`).

8. **Do not combine multiple condition updates in one row.**  
   Each row should describe **one** condition transition for the table’s single condition type.

9. **Reason is stable; Message is descriptive.**  
   - `Reason`: CamelCase enum-like code (machine-friendly, stable).
   - `Message`: short human-readable explanation (can vary).

10. **Prefer watch-driven reconcile.**  
   Use `requeueAfter` mainly for external polling or timed checks.

---

## Transition table template (single Condition type)

> Condition type modeled in this table: **`Ready`** (example).  
> Replace `Ready` with your chosen single condition type, but keep it consistent for all rows.

### Condition Transition Table — `Ready` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Ready`) | Next reconcile (Return) |
|---|---|---|---|
| `<boolean predicate over observed state>` | `<ensure/create/patch/delete/finalizer/external call>` | `Ready=<True/False/Unknown>; Reason=<CamelCase>; Message="<short>"; ObservedGeneration=<n>` | `<none | requeue | requeueAfter: <duration>>` |
|  |  |  |  |
|  |  |  |  |

---

## Example table (single Condition type: `Ready`)

### Condition Transition Table — `Ready` (single-condition table)

| Guard (Observed predicate) | Ensure (Idempotent reconcile action) | Condition transition (ONLY `Ready`) | Next reconcile (Return) |
|---|---|---|---|
| `deletionTimestamp != nil && hasFinalizer` | Delete/cleanup owned resources as needed; remove finalizer when safe | `Ready=False; Reason=Deleting; Message="Cleaning up owned resources"; ObservedGeneration=metadata.generation` | `requeue` |
| `spec invalid (e.g., replicas <= 0)` | No-op (avoid thrash); optionally emit Warning event | `Ready=False; Reason=InvalidSpec; Message="spec.replicas must be > 0"; ObservedGeneration=metadata.generation` | `none` |
| `deployment == nil` | Create Deployment with ownerRef | `Ready=Unknown; Reason=Creating; Message="Creating Deployment"; ObservedGeneration=metadata.generation` | `none` |
| `deployment exists && !deploymentAvailable` | Patch Deployment only if drift; otherwise no-op | `Ready=False; Reason=Progressing; Message="Waiting for Deployment to become Available"; ObservedGeneration=metadata.generation` | `none` |
| `all children healthy && observedGeneration == metadata.generation` | No-op | `Ready=True; Reason=Reconciled; Message="All resources are in sync"; ObservedGeneration=metadata.generation` | `none` |
