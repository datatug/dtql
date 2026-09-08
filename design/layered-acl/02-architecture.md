# Architecture and enforcement boundaries

## Ownership

```text
Browser: DataTug policy UI, query UI, structured record editor
  │ authenticated request; no caller-supplied authoritative roles
DataTug daemon: owned policies + application restrictions + token boundary
  │ OpenVaultDB HTTP, authenticated subject and actor, conditional writes
OpenVaultDB: capability grants + owned policies + adapter routing
  │ in-process, immutable verified principal context
InGitDB public DALgo adapter: DB-owned policies + storage transaction
  │ unexported enforcement access to pre/post images
Git-backed data and policy files
```

DTQL specifies values, semantics and transport contracts. DALgo implements reusable parsing, evaluation, query restrictions and write checks. Policy stores and authenticators are host-owned. DataTug's browser may hide a disabled button but only the daemon's secured path enforces its policies. A user accessing OpenVaultDB directly skips DataTug's own optional restrictions, but must still encounter OpenVaultDB and InGitDB restrictions. Do not claim that DataTug policies govern independent lower-layer clients.

The InGitDB owner is a library/adapter boundary in this MVP, not another HTTP process. OpenVaultDB cannot choose not to load its policy manifest. InGitDB's public constructor must load/validate the DB owner policy set and return a secured handle. Raw storage handles are private to trusted adapter enforcement code. Hostile Go code, repository owners and filesystem administrators are outside the in-process ACL threat boundary.

## Deterministic combination

For a complete concrete request `r`, `permit(r) = authenticated-and-capable(r) AND every required enabled policy permits(r) AND every obligation is enforced(r)`. A missing/failed/unknown required participant never counts as allow. Within each policy, retain DALgo hierarchy and principal binding semantics described in the policy format. Independent policies/layers intersect. There is no priority field allowing an upper policy to override a lower policy.

| Condition | Required behavior |
|---|---|
| Explicitly ACL-disabled owner scope | Preserve legacy access behavior and existing authentication gates; report `disabled`, not affirmative ACL allow. |
| Enabled scope with zero policies, missing manifest, incomplete configuration | Deny via `ACL_CONFIGURATION_INVALID`. Do not infer disabled from missing files. |
| Policy with no matching allow, including no subject binding | Deny via `ACL_NO_MATCH`; attribute policy, never silently skip it. |
| Multiple subject matches | Union selected rule sets inside one policy, deterministic DALgo rule precedence. |
| Multiple mandatory policies | Intersection; all applicable independent blockers collected. |
| Parent and child scopes | Existing descendant matching; more-specific rule wins inside one policy. Inherited mandatory policies are never removed by child configuration. |
| Evaluation/type/parameter error | No execution; `indeterminate`, or `deny` if another definitive denial exists. Error blocker records safe facts. |
| Required source unavailable or lacks necessary enforcement | No execution. Source unavailable or unsupported reason; visible partial coverage. |
| Unrecognized data operation, policy version or required contract field | Reject; no interpretation as a legacy unrestricted request. |

Enabled scope is **the database in MVP**. Table selection is inside policies, not a separately toggleable ACL scope. Child tables do not inherit an implicit allow. Runtime requests cannot set ACL mode, remove required owners, or supply mandatory policies. Admin can disable only through a distinct configuration privilege, outside normal policy DELETE.

## Request lifecycle

1. Authenticate ingress, resolve subject plus actor and current membership, validate token capabilities. Bind immutable principal and trusted variables. Separate query parameters from authorization variables.
2. Normalize the operation once: exact mounted database identity, canonical paths, explicit target fields, structured query or typed mutation. Reject unsupported forms before data access. Discover the required participant topology from server configuration, not browser claims.
3. Pin each owner's immutable policy snapshot/revision for the short operation. Build a DALgo evaluation plan with residual row and field obligations. `conditional` is not final permission.
4. Evaluate every available independent policy and field/operation blocker, even after a denial. Evaluate row-dependent obligations only using authorized enforcement evidence. No mutation may be used to “probe” a lower layer. Bounds are specified in the wire contract.
5. If all required obligations can be met, execute through the secured owner handles. Queries carry intersected row filters into the engine **before limit/order/paging**. Writes verify actual pre- and post-images at the mutation boundary. Denied write: no delegated mutation and no Git commit.
6. Reduce decisions, redact for the caller, and return provenance, snapshot IDs, coverage and restrictions. Execution and public Explain call the same normalization and evaluator. Explain never authorizes a later operation.

## Reads

MVP: single root collection, field projection, comparisons `==`, `In`, `>`, `>=`, `<`, `<=`, `and/or`, literal RHS or bound parameters, field order and bounded limit/offset. No joins, aggregates, functions, aliases, native SQL, dynamic identifiers, computed-field dependency inference or collection-group queries on ACL-enabled mounts. Reject unsupported paths even if a legacy opaque-query rule allows them. This is an explicitly stricter negotiated profile; legacy DALgo can retain its separate opaque capability behavior.

Row rules filter query results, so `query` may succeed with zero visible rows. That is not a list of denied hidden rows. A concrete `get`/`exists` can deny; protected and nonexistent paths must have the same public missing/denied projection where disclosure is disallowed. `query` and `get` are separate operations and Explain must not confuse them.

Field restrictions apply to explicit selection, filters, sorts and other caller references, not just result cells. A hidden field cannot become an inference channel through `WHERE hidden = ...`. Wildcard result selection is safely intersected/redacted. Explicit forbidden fields reject the request in this profile. Trusted policy predicates may refer to hidden fields inside the **owning enforcement component**; no such raw value is returned to the caller. Query alternatives use the current conservative DALgo intersection of possible field lists. The UI labels this; it must not display their union as effective access.

Each upstream row constraint is ANDed into the outgoing structured query before paging. Sending a constraint cannot grant lower-layer access. If an upstream predicate needs a field the downstream interface refuses, the adapter reports unsupported enforcement and fails closed; it must not fetch unprotected data or paginate then filter. The fixture avoids this by keeping country/status available for permitted rows.

## Writes and race safety

All operations use one typed mutation model. MVP proves key-targeted `update` with field-set assignments; insert/set/delete are specified and receive contract/adapter tests. UPDATE-by-arbitrary-WHERE, SQL mutation parsing and multi-row mutation UI are follow-up. `set` is replacement/upsert, never mapped to update; removed fields as well as added/changed fields are authorization targets. Insert checks the candidate post-image; update checks pre-image `where` plus post-image `check`; delete checks pre-image and no post-image. An update naming a forbidden field is denied even if it would set the same value.

At InGitDB and embedded OpenVaultDB, trusted pre-image reads and enforcement must occur within the existing short storage transaction. Policy reload/policy CRUD participates in the owner's snapshot admission barrier. Each operation sees one immutable policy revision; successful edits affect operations admitted afterwards. No long-running transaction protocol is added.

Across DataTug → OpenVaultDB HTTP, the current buffered DALgo driver does **not** make a prior read atomic with a later batch. For a row-dependent DataTug write, require a revision-bearing authorized point read and a mutation `ifDataRevision` checked under the OpenVaultDB/InGitDB transaction lock. The revision is owner-issued and covers the **whole record**, not only visible fields; it is not a client content hash. The candidate post-image is computed from that same evidence. If hidden fields needed by DataTug are unavailable, conditional writing is unsupported and fails closed. Internal InGitDB checks do not require granting public read authority, but the MVP remote DataTug conditional-write profile does require such evidence. Write-only unconditional grants remain possible.

DataTug pins its policy/principal snapshot through the outgoing request and response. OpenVaultDB independently pins its snapshot, reauthenticates, and InGitDB evaluates current owner policy at execution admission. A data-revision mismatch returns 409 `DATA_REVISION_CONFLICT`, performs no mutation and requires re-read/re-evaluation; do not retry using the old decision. A previous Explain revision is advisory, not an access token. Policy edits changing lower policy between Explain and execution are naturally rechecked. This is per-owner short-operation snapshot semantics, not a globally simultaneous policy snapshot or distributed ACID guarantee.

Triggers, generated fields or schema transformations affecting security fields must run before the final post-image ACL check, within the same short transaction. If the provider cannot expose the final candidate or demonstrate equivalent atomic checks, advertise the operation unsupported. MVP fixtures contain none of these transformations.

## Collecting blockers without unsafe execution

Provide an internal read-only authorization inspection path behind the same authentication and disclosure checks. OpenVaultDB invokes its own and embedded InGitDB evaluation against one safe evidence snapshot. DataTug invokes it even when DataTug has a static column denial, then appends DataTug decisions. Sending a denied mutation for execution is forbidden.

A regular authenticated data caller may invoke self-inspection only for the same bounded resources/operation they could submit; the response uses ordinary diagnostic redaction. Enhanced Explain and another-principal simulation require separate privileges. An upper component is not automatically entitled to lower-layer raw images or detailed reasons. For the E2E diagnostic identity, explicitly grant safe policy-reference diagnostics at each owner. InGitDB may evaluate a row it refuses to reveal and return its row blocker; upper row evaluation can remain unevaluated while upper static column blockers are still returned. Known denial remains definitive despite partial coverage.

Coverage distinguishes evaluated facts, redacted facts, deferred row predicates, unavailable evidence, and bounded truncation. “All blockers” means all independent blockers safely evaluable within advertised limits, never all hidden rows in a query or all unreachable services. There is no global search for a minimal unsatisfied policy set. A losing allow/deny rule within one policy's precedence is not an independent blocker.
