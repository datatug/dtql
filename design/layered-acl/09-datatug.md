# DataTug policy UI, dry run and Explain Access

## Server and client responsibilities

Reuse DataTug CLI `pkg/accesspolicies` loading, variable parsing and secured read orchestration. Replace its English-derived binding attribution and field-union presentation with structured evaluator output. Move generally applicable field-reference enforcement into DALgo. Expose owner provider APIs through the existing daemon/connection architecture; do not implement an ACL evaluator in TypeScript. Preserve trusted local CLI `--no-policies`/principal simulation behavior separately; HTTP clients cannot disable mandatory server policies or assert arbitrary roles.

The daemon owns DataTug policies in a configured per-connection manifest store, using the existing policy directory loader as a starting point with stricter owner-configured path/revision rules. A user-provided CLI directory is not automatically a server-admin policy store. Extend authenticated middleware to bind principal context; replace request-path `context.Background()` loss for protected execution. Policy CRUD persists at the owner, never in browser storage alone.

## Policy workspace

Connection → Access opens visible layers in route order. Show owner/product, database, enabled/disabled/unavailable state, revision and visible policy names. Multiple instances of the same product have distinct source labels. A non-public lower layer is “additional restrictions may apply” where even its identity is private; do not show an invented green allow row.

Selecting a readable policy shows subjects (users/roles/groups), operations, paths, row admission/post-image conditions and fields. Show editability from owner capabilities and editor-format support independently. Custom/read-only policies stay visible with provenance and explanation support. Form editor MVP handles the common fixture shape, stable rule IDs, subject bindings, comparisons/AND/OR and exact schema fields; it must preserve the full AST of untouched supported structure. If fidelity cannot be guaranteed, show read-only instead of silently rewriting it. Optional raw YAML editing uses the same parser/validator.

Create/update/delete are privileged owner operations. Save validates locally for feedback and authoritatively at the owner, sends `If-Match`, and waits for the returned new revision before updating UI state. Conflict shows server revision and preserves the draft for comparison; no automatic overwrite or broadening merge. Delete requires owner permission and never turns ACL off. Changing “denied columns” in the form serializes an explicit allowed-field policy; label that new fields are denied until granted. Never promise a native deny-list DSL absent from the contract.

After save, reload the owner document and rediscover revisions, clear cached Explain/results tied to old revisions, and label preexisting result tabs stale. Reconnect/restart must obtain policies again from the owner. Policy text/source locators are not analytics payloads. Render labels and policy descriptions as text, not HTML or clickable arbitrary URLs.

## Explain and dry run

Use one UI with three choices: **Policy conditions (no rows read)**, **Specific row IDs**, **Sample top N readable candidates**. With no rows, show structured admission predicates where disclosable and small, otherwise opaque restrictions marked private/too large. Never imply that omitted conditions mean unrestricted access. Per-layer result vocabulary: Allowed, Conditional, Denied, Could not evaluate, ACL disabled. Effective result uses the C2 reducer.

Default subject is authenticated self and actual resolved memberships. Authorized Explain users can select another stable principal; server resolves roles/groups. Hypothetical role/group editing is a separately privileged simulation and visibly labeled; it cannot become execution context. Operation selection distinguishes query/get/exists and insert/set/update/delete. A write Explain requires candidate data/changes to evaluate post-image checks; selecting only “update” and a path gives static/conditional results, not a fabricated final answer. WP1 should make absent mutation content a plan-only operation descriptor; inspect of an update without changes rejects 400.

For specific keys, show per-path/per-column blockers grouped by owner, preserving policy references. Ordinary callers get whatever safe projection each owner permits. Private rows are never fetched into the browser for Explain. For sampling, show selection/filter/order, requested limit, evaluated count and “sample only”; no claim about every matching row, no hidden row count. A zero-row sample is inconclusive. Users can run a new explicit sample, not an unbounded “explain all” action.

```text
customers/123   UPDATE credit_limit, status
InGitDB       Denied        tenant-access / update-tenant
OpenVaultDB   Denied        financial-fields / update-fields
DataTug       Denied        workflow-fields / update-fields
Effective     Denied
Some additional row checks were not evaluated because evidence is private.
```

This display requires granted reference diagnostics; ordinary users may see generic source-level denial instead. No hard-coded English explanation arrives from DTQL; DataTug localizes reason codes and renders safe facts. Each response displays policy revisions and whether it was plan/inspect/sample, hypothetical and complete/partial. Changed revisions invalidate prior displays. Explain is explicitly advisory until the actual operation is reauthorized.

## Real read and write

Reuse the query result grid for a supported DTQL query through the daemon → OpenVaultDB connection. ACL-enabled native/opaque query tabs are rejected unless a separately supported future enforcement profile exists. Do not rely on the browser's SQL convenience parser to establish accessed tables/fields. Keep the existing query builder read-only by construction.

Add a small record-edit flow on an authorized row: choose exact fields, edit scalar values, optionally Explain the candidate, submit a key-targeted typed UPDATE with the owner data revision. The server obtains/validates evidence and binds its policy snapshot; stale record → conflict and reload, never silent retry. Display returned blockers across layers in one response. A write may be denied even when buttons were enabled or Explain previously allowed; the UI must handle this as normal changed-state behavior.

## UI acceptance

Accessible keyboard form and tables, clear textual statuses (not color alone), progress/cancel for dry run, sanitized errors, and retained draft on failed save. Tests use signals/OnPush-compatible behavior through Nx targets determined from installed project configuration. E2E must use the real daemon/server/adapter, not only mocked policy responses. Use test-only fixture credentials via server provisioning, never bake owner tokens into the frontend.
