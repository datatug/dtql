# Executable acceptance and E2E test plan

These are implementation acceptance specifications, not tests already implemented or passed. Every test uses the frozen C1–C6 contracts; assertions compare structured codes/provenance, not English messages. Fixture setup uses isolated temporary directories and loopback services. No production databases or external identity service required.

## Reference fixture

Start the real DataTug daemon, OpenVaultDB HTTP server and local InGitDB adapter, with a real Git repository and persisted policy stores at all three owners. Drive the real browser policy workspace/query grid/record edit via the app's E2E runner. Record exact component SHAs/versions, policy revisions and test artifact hashes. Policy fixtures explicitly use `visibility:public` where non-admin reference diagnostics are asserted; privacy tests use private policies.

Principals in realm `acl-e2e`: `user-alice` (role `crm-editor`, group `crm-team`, trusted `tenant=A`), `user-bob` (no editor binding), `policy-admin` (owner-scoped policy admin/CRUD grants), `policy-editor` (delegated CRUD only for selected public policies), actor `datatug-client` (delegated to Alice, read/update scope only). Also provision direct OpenVaultDB Alice token and direct InGitDB trusted principal context. Admin rights per owner are explicit. Diagnostic Alice has `access:diagnostics` for safe references at all owners; ordinary Alice token omits it. No owner token in browser.

Rows have fields `id,tenant,country,status,name,email,credit_limit`. ID is the record key and visible for authorized rows; update of ID is unsupported.

| ID | tenant | country | status | name | email | credit_limit |
|---|---|---|---|---|---|---|
| 101 | A | IE | active | Ada | ada@example.test | 100 |
| 102 | B | IE | active | Bea | bea@example.test | 200 |
| 103 | A | US | active | Cal | cal@example.test | 300 |
| 104 | A | IE | paused | Dee | dee@example.test | 400 |
| 105 | A | IE | active | Eli | eli@example.test | 500 |

Each layer has separate mandatory **row-access** and **field-access** policies. This is intentional reuse of DALgo intersection: unconditional field checks remain independently evaluable when a different row policy denies. Do not attempt to encode conditional `deny fields` rules.

| Owner | Row policy: `get/exists/query/update` on `/customers` | Field policy read | Field policy update |
|---|---|---|---|
| InGitDB (`ingit-crm`) | `tenant-access`: allow where tenant == trusted tenant; check defaults where | All fixture fields | name,email,credit_limit,status |
| OpenVaultDB (`ovdb-crm`) | `country-access`: allow where country == IE; check defaults where | All fixture fields | `financial-fields`: name,email,status |
| DataTug (`datatug-crm`) | `workflow-access`: allow where status == active; check defaults where | All fixture fields | `workflow-fields`: name,email,credit_limit |

All row policies use the bound `editor` rule set with user/role/group matches; selecting multiple matches deduplicates. Field policy read rules are unconditional, update field rules unconditional. No policy grants other data actions in the baseline; insert/set/delete tests temporarily add explicitly scoped rules. Add a read-only application `Policy` named `application-boundary`, public to the test principal, allowing only customer reads/update. It implements pure inspection, is mandatory and noneditable; no external service is needed.

## Core end-to-end flow

### T01 — Real query, conditions and concrete Explain

Given baseline policies, when DataTug runs DTQL `from: {name: customers}` ordered by ID, then actual result IDs are exactly `[101,105]`. Record query execution at the storage boundary to prove tenant AND country AND status constraints apply before limit. Re-run with limit 1: `[101]`, not an empty page caused by post-page filtering.

When plan dry run runs with no row IDs, then no adapter row read/query occurs, result is conditional with row admission restrictions for each owner, and safe expressions or private/too-large references. Inspect `get /customers/101` allows; inspect `get /customers/102` denies InGitDB; actual gets match at unchanged snapshots. A query filtering to 102 succeeds with zero visible rows; Explain `query` must not falsely claim a point-get allow or list hidden rows.

### T02 — Real successful UPDATE

Given authorized revision-bearing read of 101, when the browser changes `name` to `Ada Updated`, explains the same update and submits it, then Explain and execution allow, exactly that field changes, owner data revision changes, and one data Git commit records the mutation. All three row policies and field policies participate. Verify persisted file contents independently; an HTTP 200 alone is insufficient.

### T03 — Independent enforcement and bypass attempts

Given unchanged policies, when DataTug permits a lower-denied operation, then OpenVaultDB/InGitDB still deny. Direct OpenVaultDB updates of 102 fail at InGitDB; direct OpenVaultDB update `credit_limit` on 101 fails at OpenVaultDB. Direct InGitDB update of 102 fails at InGitDB. Direct OpenVaultDB change of `status` on 101 may succeed if its own/lower post-image constraints permit it: this correctly demonstrates DataTug policy scope, not a lower-policy bypass. Restore fixture after each branch. New `context.Background()` and transaction/reader handles cannot discard mandatory owner policies. An owner token has no data ACL override on an enabled mount.

### T04 — Deterministic composition

Given user/role/group all bind editor, then authority is not multiplied. Permute policy load order and subject membership order: same effective restrictions/blocker sets after canonical sorting. More-specific allow reopens same-policy parent deny; same-specificity deny wins; a separate policy deny remains final. New no-match principal Bob denies despite everyone/other unmatched permissions. Table no-match denies before data access.

### T05 — Edit, persist, reconnect, rediscover, execute

Given `policy-editor` can update the public OpenVaultDB financial-fields policy, when the UI adds credit_limit to its update allow-list using current ETag, saves, reconnects and restarts OpenVaultDB, then rediscovery returns the new persisted revision and AST. Explain update credit_limit on 101 now allows; actual UPDATE succeeds while status remains denied at DataTug. Restore policy, reconnect and verify denial returns. Create/delete an extra narrowing test policy through UI and verify behavior and revisions after reload. Unauthorized owner or stale ETag cannot mutate it; removing the last mandatory policy cannot disable ACL.

### T06 — Read-only and custom policy representation

Given application-boundary is visible and noneditable, then UI displays its source and contribution, with no save/delete controls. Direct CRUD API attempts fail. Make its pure evaluator deny in a controlled fixture: Explain and real operation both deny with application provenance. A nonserializable custom policy is not omitted or rewritten. Non-pure inspection is explicitly partial/unsupported.

### T07 — Three blockers from one intended UPDATE

Given diagnostic Alice requests update 102 setting credit_limit=10000 and status=vip, when DataTug runs inspect or handles the denied write, then return at least these three independent safe facts in one structured response:

1. InGitDB tenant-access: `ACL_ROW_PREDICATE_FAILED`, row scope/where.
2. OpenVaultDB financial-fields: `ACL_COLUMN_DENIED`, credit_limit.
3. DataTug workflow-fields: `ACL_COLUMN_DENIED`, status.

Each fact retains its exact owner/layer, policy revision and rule reference. Upper row predicates needing unavailable pre-image may be unevaluated, with explicit partial coverage; this does not remove the known three denials or fabricate additional predicate failures. No row values are disclosed. Data bytes and Git HEAD remain unchanged. Repeat with ordinary diagnostics: outcome still denies, hidden facts coalesce safely.

### T08 — Dry run purity and bounded completeness

Given instrumented storage/provider callbacks, plan makes zero data reads; inspect reads only supplied bounded keys; sample issues one bounded candidate selection plus bounded assessments; all perform zero mutation/trigger/Git commits. Upper static deny still requests lower safe evaluation. A required unavailable provider gives partial coverage and deny/indeterminate, never allow. Timeout/truncation is visible. Execution cannot use a dry-run response as permission. Malicious custom callbacks cannot execute through a purity-undeclared inspect path.

## Security and contract cases

| Test | Given / When | Required observable outcome |
|---|---|---|
| T09 Private policy disclosure | Private policy; reader/editor/Explain/diagnostics non-admin requests text, ID probe, conditions, blockers and sampling | Generic denial/opaque restriction only; no name, ID, text, predicate, hidden count, revision or existence oracle. Same owner policy admin can see it. OpenVaultDB admin cannot see private InGitDB policy without InGitDB admin grant. |
| T10 Field inference | Query selects, filters, sorts, aliases or computes on a hidden field | Rejected before query; wildcard selection redacts; no filter/order leakage. Unsupported joins/functions/native SQL fail closed. |
| T11 Changed state | Explain permits; change row/policy/membership before actual mutation | Actual current policy/membership denies, or data CAS conflicts; no write. Include hidden-field-only change to prove whole-row revision, not projected hash. |
| T12 Request spoofing | Browser sets principal/roles/tenant or uses unrelated issuer token/client | Reject/ignore untrusted identity fields per contract, never grant. CORS/session and missing/expired token tests fail safely. |
| T13 Actor and membership | Subject permits but actor scope denies; remove role/group after dry run; omit context | Capability/principal blocker; current resolver enforced; missing required identity fails closed. |
| T14 Policy integrity | Concurrent edits, invalid candidate set, symlink/traversal, missing enabled manifest, failed Git commit | 412/validation error and prior state unchanged; no disabled fallback; restart preserves old or new complete set only. Non-admin cannot change private/public classification. |
| T15 Parsing and limits | Round-trip supported documents and malformed/oversized/deep/duplicate YAML/JSON; unknown versions | Canonical idempotence and decision equality for valid inputs; bounded validation rejection for invalid. No dropped unknown fields. |
| T16 Generic writes | Insert candidate wrong tenant; update moves tenant; set drops protected field; nested set changes sibling; multi-target one denied | Deny and no mutation; correct where/check/fields facts. Delete checks pre-image. No trigger/schema changes before final authorization. |
| T17 Adapter capabilities | Non-DTQL SQLite behind OpenVaultDB; opaque native ACL source; adapter lacks revisions | OVDB policies enforced on SQLite smoke; opaque lower denial stays authoritative; conditional remote writes unsupported without atomic revision capability. |
| T18 Top-N sample | Sample 1/2 visible candidates, no IDs; targeted update includes forbidden columns | Deterministic IDs/order, evaluated count ≤N, denied write columns included; no writes; sample-only flag. Zero sample conditional. No inference about hidden rows; target principal's broader read rights do not widen requester's visibility. |
| T19 Condition size/privacy | No rows, one private predicate and one expression over disclosure limit | Opaque private/too_large references; result remains conditional; no weakened/silently cut predicate; private policy only visible to owner admin. |

## Test layers and run evidence

WP1 conformance vectors specify exact request/decision examples; WP2 unit/property/fuzz tests; WP4–6 real adapter/HTTP tests; WP7 UI interaction tests; WP8 full browser vertical flow. Reuse existing DALgo tests and InGitDB temporary-repo fixtures. Version-pin all Go/TS modules in an E2E manifest; CI artifacts include sanitized requests/responses, layer revision traces, Git before/after and browser failure screenshots. Test assertions must verify all 13 mission proofs: T03(1,2,12), T04(3), T05/T06(4,5,6,13), T01/T02(7,8,9), T07(10,11).

Pass gate: all mandatory T01–T19 checks at supported capabilities, no skipped reference InGitDB/UI cases, no credentials/row secrets in artifacts, and no forced best-effort adapter mode. Broader cloud/native sources may be explicitly out of profile, not silently skipped while claiming conformance.
