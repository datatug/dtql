# Independent Architecture Review — Layered ACL MVP design packet

**Reviewer slot:** Opus (independent). **Packet:** README + 01–14 + audit-snapshots.json + reviews/rubric.md + reviews/source-evidence.md, as frozen. **Date:** 2026-09-08.
**Basis:** packet text only, plus the source excerpts supplied in `reviews/source-evidence.md`. I did not open the repositories, did not run tests, and did not read any other review.

---

## 1. Verdict

**Ready with changes** — the architecture is sound and unusually well-grounded in the existing code, but three `high` findings (O-01, O-02, O-03) are unresolved *contract* gaps, not implementation details, and must be closed in the documents before C1–C6 are frozen and packages are dispatched.

Rationale. The core composition rule (`permit = authenticated ∧ every required policy permits ∧ every obligation enforced`, intersection only, missing participant ≠ allow) is simple, checkable and correctly refuses priority/override fields. The packet reuses DALgo's shipped hierarchy, condition vocabulary, codecs and secured wrappers rather than inventing a DSL, and its current-state audit is honest about what is placeholder, stale or duplicated. The dry-run purity split (plan reads nothing / inspect reads supplied keys / sample reads bounded authorized candidates) and the `allowed`-only-for-complete-allow rule are the right shape. The reference fixture is internally consistent (I checked T01 `[101,105]` and the T07 three-blocker derivation against the fixture tables and they hold).

What is not yet decidable by an implementer: (a) how a layer's enforcement code obtains an unredacted pre-image through *another* owner's secured facade (O-01); (b) whether the structured blocker taxonomy is allowed to be a row-existence/row-attribute oracle and under which grant (O-02); (c) how an upper layer's row predicate survives the HTTP hop without being mistaken for a caller field reference, and whether the upper layer must verify it was applied (O-03). Each of these is load-bearing for the packet's central claim of independent layered enforcement, and each is currently answered only by prose that gestures at the problem ("nested secured remote wrappers complicate that distinction", "the fixture avoids this by keeping country/status available").

No `critical` finding: I found no path where the packet as written plausibly yields cross-boundary unauthorized data access with no viable contract. The high findings fail closed or are contained in-process; their risk is that implementers will resolve the ambiguity divergently and unsafely.

---

## 2. Findings

### Summary table

| ID | Sev | Dimension | Blocks C-freeze | Summary |
|---|---|---|---|---|
| O-01 | high | layered enforcement / DALgo boundary | **yes** (C4, C5) | No contract for how nested secured handles supply unredacted pre/post-image evidence |
| O-02 | high | security / privacy | **yes** (C2) | Blocker taxonomy is an existence + attribute oracle; missing-row mapping undefined; 404/403 split |
| O-03 | high | layered enforcement / correctness | **yes** (C2, C6) | Cross-layer predicate push-down: no provenance channel, no enforcement attestation |
| O-04 | medium | correctness / C1 semantics | **yes** (C1, small) | Read field bounds under terminal allow + narrower conditional allow ambiguous vs code |
| O-05 | medium | correctness / migration | **yes** (C1) | Qualified rule IDs change legacy precedence tie-breaks; equality claim may be false |
| O-06 | medium | security / threat model | no (decision) | Git write access to the data repo == InGitDB policy admin |
| O-07 | medium | discovery / privacy / C1 | **yes** (C1, small) | `metadata.visibility` has two sources of truth; PUT-omission semantics undefined |
| O-08 | medium | dry run / disclosure | **yes** (C2, small) | `omissionReason` lacks `not_authorized`; `fields`/`policyRef` gating unstated |
| O-09 | medium | correctness / usability | no | Upper policies silently unenforceable when lower layers hide referenced fields |
| O-10 | medium | contract completeness | **yes** (C2/C5, small) | Batch/multi-target write semantics over HTTP undefined (per-item CAS, atomicity, envelope) |
| O-11 | medium | testability | no | Acceptance exercises only string equality, one realm, one hop |
| O-12 | medium | schema boundary | no | Schema discovery/DDL outside the policy model on ACL-enabled mounts |
| O-13 | medium | unnecessary complexity / MVP scope | no (human) | First slice carries three deferrable subsystems; 8-stage serial critical path |
| O-14 | low | policy expressiveness | no | No `!=`/`not in`/`not`, no predicate deny; "everything except X" inexpressible |
| O-15 | low | consistency semantics | no | Policy-snapshot pin window across a request is undocumented for operators |
| O-16 | low | contract clarity | no | `restrictions[].enforced` undefined and dangerously readable as "not applied" |
| O-17 | low | acceptance completeness | no | "13 mission proofs" referenced but never enumerated in the packet |
| O-18 | low | operations / feasibility | no | No lock-contention/throughput criteria; write path serializes on one exclusive repo lock |
| O-19 | low | interop | no | No version negotiation for `dtql.org/authorization/v1` beyond exact-string match |
| O-20 | low | identity | no | Bootstrap/owner principal has no stable `(realm,kind,id)`; actor-vs-subject denial unattributable |

### Detail

**O-01 — high — Nested secured handles have no evidence contract. Blocks C4/C5.**
*Evidence:* 02 "Raw storage handles are private to trusted adapter enforcement code"; 07 "return a secured DALgo facade"; 05 "Strengthen wrappers without requiring caller-visible read permission for in-process enforcement reads"; 01 already flags it ("nested secured remote wrappers complicate that distinction"); `access/write.go:79-94` obtains the pre-image via `s.session.(dal.ReadSession)` — i.e. whatever session it was constructed over.
*Concrete failure:* OpenVaultDB's policy layer is attached **outside** InGitDB's secured facade (08: mount opens InGitDB via its secured constructor, "then attaches OpenVaultDB's own policy layer"). To enforce `country-access` on `update /customers/101`, OpenVaultDB's `securedWriteSession` must read the pre-image — through InGitDB's secured session. Two outcomes, both wrong: (i) it reads through the caller-visible path and gets an InGitDB-redacted or denied row, so OpenVaultDB's `where`/`check` evaluates against an incomplete image (false deny, or worse, a `check` that passes only because a field is absent — note C1's "missing fields do not match" makes an absent field silently fail a `where` and silently satisfy a negated intent); (ii) an implementer gives the outer wrapper a raw handle, and InGitDB's field-level read restrictions are bypassed by a peer layer's enforcement code with no attestation of who may do so. The packet defines this contract only for the *remote* case (whole-record revision, fail closed) and leaves the in-process case, which is the MVP's actual composition, undefined.
*Action:* Add to C4 an explicit enforcement-evidence interface (unredacted pre-image + whole-record revision + final candidate post-image), obtainable only by attested trusted enforcement callers, with: required wrapper nesting order, a rule that all layers in one operation share a single evidence read inside the one storage transaction (avoiding N reads and N-way skew), and a rule that evidence never crosses an owner boundary except as the already-specified revision+authorized-read profile. Add WP3/WP4b tests asserting that an outer layer's pre-image is unredacted *and* that the caller never receives it.

**O-02 — high — The blocker taxonomy is an existence and attribute oracle; no grant governs it. Blocks C2.**
*Evidence:* 04 defines `ACL_ROW_PREDICATE_FAILED` ("actual inspected pre-image fails row admission"); 02 requires "protected and nonexistent paths must have the same public missing/denied projection where disclosure is disallowed"; 10 grants `access:diagnostics` = "policy/rule references and safe blocker facts"; `access/write.go:135-142` already emits a *different* denial for a missing row than for a non-matching row; `pkg/server/httperr.go:30-31` maps `core.ErrNotFound` to 404 `not_found` while an ACL denial will be 403 `access_denied`.
*Concrete failure:* Diagnostic Alice inspects `get /customers/999` (absent) and `get /customers/102` (tenant B). Under the current text, 999 plausibly yields not-found/`ACL_NO_MATCH` and 102 yields `ACL_ROW_PREDICATE_FAILED` with `slot:where` on the visible `tenant-access` policy — so she learns (a) row 102 exists and (b) its `tenant` ≠ `A`, for a row she may not read. `access:diagnostics` was defined as a *policy-reference* grant, not a row-existence grant, yet it silently confers both. Other-principal Explain amplifies this: a helpdesk operator with `access:explain` + `access:diagnostics` can enumerate row attributes of rows they cannot read, one predicate at a time.
*Action:* (i) Separate "may learn policy/rule identity" from "may learn that a row exists and failed a predicate" — the latter belongs with `access:inspect-protected` or a new explicit tier; without it, row-scoped facts must coalesce to `ACCESS_DENIED`. (ii) Define the canonical code and HTTP projection for "target row missing under a conditional grant" and require it to be byte-identical to the denied-existing-row projection on ACL-enabled mounts (this contradicts the current 404/403 split in `httperr.go`, which WP5b must change deliberately). (iii) Add a T09 sibling test that asserts indistinguishability across missing/denied/hidden for each disclosure tier, including other-principal inspect.

**O-03 — high — Cross-layer predicate push-down has no provenance channel and no enforcement attestation. Blocks C2/C6.**
*Evidence:* 02 "Each upstream row constraint is ANDed into the outgoing structured query before paging" and "If an upstream predicate needs a field the downstream interface refuses, the adapter reports unsupported enforcement and fails closed"; 05 "A `QueryPlan` holds original caller references separately from owner-injected predicates"; 04 wire query is `{format:"dtql-yaml", text, parameters?}` — a flat text blob with no provenance field.
*Concrete failure (a), false deny:* DataTug ANDs `status == active` into the DTQL text sent to OpenVaultDB. At OpenVaultDB the injected predicate is indistinguishable from a caller reference, so OpenVaultDB's own hidden-field reference check (the DataTug check being migrated into DALgo per 01 §2 / 05) rejects it if `status` is outside that principal's OpenVaultDB read field allow-list. The packet's `QueryPlan` provenance solves this in-process and is lost at the first HTTP hop. The fixture cannot expose this because every layer's read field policy is "All fixture fields".
*Concrete failure (b), unverified delegation:* DataTug's own row policy is enforced *only* by OpenVaultDB/InGitDB's query engine. Because filters must be applied before paging, DataTug cannot post-filter without breaking pagination, so a lower layer that drops, mistranslates or partially applies the injected predicate causes DataTug to return rows violating DataTug's policy, undetectably. 02 correctly says "Sending a constraint cannot grant lower-layer access" but never states the converse trust assumption.
*Action:* Add an owner-attributed `constraints[]` channel to the C2 request (each entry: originating `layerId`, condition in C1 shape), specified as (i) exempt from caller-reference checks, (ii) never privilege-granting, (iii) required to be echoed in the response as an applied-constraint attestation the upper layer verifies before publishing results; on missing/partial attestation, fail closed with `ACL_ENFORCEMENT_UNSUPPORTED`. Additionally require cheap defence-in-depth re-verification of returned rows where the referenced fields are visible to the upper layer. Add a negative fixture in which an upper-layer predicate field is hidden by a lower layer.

**O-04 — medium — Read field bounds under a terminal allow plus narrower conditional allows are ambiguous and may contradict the code. Blocks C1 (small).**
*Evidence:* 03 §4 "An unconditional allow makes row eligibility unrestricted, though field restrictions remain. Query field bounds conservatively intersect all possible deciding allow alternatives"; `access/policy.go:276-281` — when `write.Terminal != nil` the read decision returns with `Residuals` dropped and only the terminal rule named. `access/fields.go` and `session.go` were not in the packet, so I cannot confirm what field set is then applied.
*Concrete failure:* Policy with `allow(where tenant==$tenant, fields:[name])` at `/customers` plus a parent `allow` (unconditional, no `fields`). Spec text (conservative intersection) implies effective read fields `[name]` for all rows — stricter than the terminal grant. The code path plausibly yields all fields. A UI showing one and an evaluator enforcing the other is exactly the class of divergence C1 exists to prevent.
*Action:* State the rule precisely in 03 (I recommend: field bounds are the union over the alternatives that can decide a given row, and for `query` the conservative intersection of possible deciders including the terminal — whichever is chosen, say it), verify it against `access/fields.go`/`session.go` before freezing C1, and add golden vectors for this shape across `get`, `query` and `update`.

**O-05 — medium — Qualified rule IDs change legacy tie-breaks; the "equal decisions" migration claim may be unprovable. Blocks C1.**
*Evidence:* `access/policy.go:361-373` — final tie-break is `left.name < right.name`; 03 "Prefix rule IDs by rule-set ID and sort deterministically" and "Migration tests must prove equal decisions for legacy supported constructs, including this tie behavior."
*Concrete failure:* Two rule sets both bound to the caller, each with an equal-depth, equal-literal, same-effect conditional `allow` on `/customers` but different `fields` allow-lists. Order decides which alternative decides a write (`write.go:143-167` returns on the first alternative whose `Where` holds), hence which field allow-list is enforced. Renaming `rule` → `editor/rule` reorders them, so decisions differ. The packet asserts equality must be proven; for this shape it is false, not merely untested.
*Action:* Make it an explicit C1 decision: either keep the legacy unqualified name as the sort key (qualification used only for attribution), or accept a documented behavior change gated on `apiVersion: dtql.org/access/v1` with a conversion-time warning and differential fixtures containing cross-rule-set name collisions. Separately, consider adding an explicit `priority`-free but *stable declaration-order* tie-break so that renaming a rule cannot change authorization at all — the current "editor must warn on rename" mitigation is weak for a UI that will create rules.

**O-06 — medium — Repository write access is de-facto InGitDB policy admin.**
*Evidence:* 07 stores policies at `.ingitdb/access/policies/*.yaml` inside the data repository; 07 "A malicious repository/OS owner can change all files and is outside this ACL boundary"; `ingitdb-specs` contains GitHub app/PR permission specs, i.e. a collaboration model where non-administrators obtain repo write.
*Concrete failure:* In any deployment where data changes arrive by Git push or merged PR — the product's stated direction — a data contributor edits `.ingitdb/access/policies/tenant-access.yaml`, and on the next reload/reconnect the server adopts it without passing owner policy-CRUD authorization, revision CAS or the last-mandatory-policy guard. The exclusion clause is technically true but hides a product-level consequence.
*Action:* State explicitly that "write access to the data repository is equivalent to InGitDB policy administration", and constrain the roadmap: policies in a protected location/branch excluded from the PR write path, or a signed manifest, or an out-of-repo policy store, before any cloud/GitHub-backed adapter is enabled. Add it to 10's boundary list and 13 as a human decision.

**O-07 — medium — `metadata.visibility` has two sources of truth. Blocks C1 (small).**
*Evidence:* 03 metadata "optional `visibility: private|public`, default public"; 03/10 "The owner persists and attests the effective classification and rejects a non-admin replacement that changes it."
*Concrete failure:* A delegated editor `PUT`s a valid document that omits `visibility`. Is that a declassification attempt (reject), a no-op (accept), or a reset to default public (silent leak of a private policy)? Also, canonical idempotence (D16) requires a deterministic canonical form: does canonical output carry the document's value or the owner's attested value, and does export/import round-trip through a *different* owner preserve or reset it?
*Action:* Make the owner store authoritative; treat the in-document field as a create-time hint that is validated and then stripped from the canonical form (or required to match, with mismatch rejected as a precondition failure); define omission on `PUT` as "unchanged". Add to T14/T15 fixtures.

**O-08 — medium — `omissionReason` enum is incomplete; `fields` and `policyRef` gating unstated. Blocks C2 (small).**
*Evidence:* 04 `omissionReason:private|too_large|unsupported`; 10 "Predicate literals still require policy-read/disclosure permission"; 04 restriction shape includes `policyRef?`, `expression?`, `fields?` with only "Resolved secret values/predicate text omitted unless explicitly disclosable."
*Concrete failure:* An ordinary caller with no `policies:read` runs plan mode against a **public** policy. The server must omit the expression but has no correct reason value; implementers will emit `private`, which conflates "you lack a grant" with "this policy is classified private", degrading the very classification signal D10 introduced.
*Action:* Add `not_authorized`. State that `expression`, `fields` and `policyRef` are independently gated (a caller may legitimately see a policy reference but not its predicate, or a field count but not names), and define the disclosure matrix once in 10 rather than in prose across 04/09/10.

**O-09 — medium — Upper-layer policies become silently unenforceable when lower layers hide their referenced fields.**
*Evidence:* 02 "If hidden fields needed by DataTug are unavailable, conditional writing is unsupported and fails closed"; 02 "The fixture avoids this by keeping country/status available for permitted rows."
*Concrete failure:* An operator sets OpenVaultDB `financial-fields` to hide `credit_limit` from the `datatug-client` actor, while a DataTug policy uses `check: credit_limit < 1000`. Every DataTug write now returns `ACL_ENFORCEMENT_UNSUPPORTED` with no authoring-time signal, no diagnosis pointing at the lower field policy, and no UI affordance — a self-inflicted outage that looks like a bug. The layered model's practical usability depends on upper layers being able to *see* what their predicates reference, and nothing validates that.
*Action:* At policy create/update, validate each referenced field against the configured lower stack's advertised field capability for the intended bindings, and reject or warn with a specific code; surface it in the WP7 form editor; add a T-case.

**O-10 — medium — Batch / multi-target write semantics over HTTP are undefined. Blocks C2/C5 (small).**
*Evidence:* 08 "accept `ifDataRevision` on key-targeted mutation/batch operation"; T16 requires "multi-target one denied → deny and no mutation"; 01 notes `dalgo2openvaultdb` uses buffered writes and existing OpenVaultDB has batch writes.
*Concrete failure:* A two-item batch where item 1's revision matches and item 2's does not. Is `ifDataRevision` per item or per batch? Is the whole batch executed inside one InGitDB `RunReadwriteTransaction` (`database.go:135`)? Does the client get one 409 or a per-item result array? The taxonomy has no per-item error envelope, and `restrictions`/`blockers` key on `operationId`, which exists in `/evaluate` but not in the execution batch endpoint.
*Action:* Either (i) specify: ACL-enabled batches execute in one lower-owner RW transaction, preconditions are per item, any failure aborts all, and the 409/403 envelope carries per-item `operationId`-keyed blockers; or (ii) explicitly exclude batch from the ACL-enabled MVP profile and have `GET /layers` advertise it as unsupported. Do not leave it to WP5b.

**O-11 — medium — Acceptance exercises a fraction of the C1 semantic surface.**
*Evidence:* 11 fixture — every policy predicate is a string equality (`tenant==`, `country==`, `status==`), one realm `acl-e2e`, one OpenVaultDB hop, no nested fields, no path captures, no `In`/`or`/ordering; contrast 03's specified semantics (numeric no-coercion, ±2^53−1 bound, UTF-8 lexical ordering, explicit-null equality only, missing ≠ match, `path.<capture>`, dotted nested fields) and 06's cross-realm mapping requirement.
*Concrete failure:* An InGitDB query engine that compares YAML-typed values as strings would push down `credit_limit > 100` differently from `condeval`'s in-memory evaluation, producing rows the policy forbids — and T01–T19 would all pass. Likewise a cross-realm mapping bug is untestable in a single-realm fixture, and `layerId` distinctness for two hops is asserted in 04 but never tested.
*Action:* Add to WP1 an adapter predicate-semantics conformance vector set (numeric/string, null vs missing, type mismatch, ordering, `In`) that every adapter must execute (extend T17), and extend the reference fixture with one numeric-ordering policy, one `In`, one null/missing case, one nested field, one `{capture}` path and one cross-realm binding.

**O-12 — medium — Schema discovery and DDL are outside the policy model on ACL-enabled mounts.**
*Evidence:* C1 leaf operations are the eight data verbs plus `truncate`; 08 lists "schema/collection operations" among escape routes to review but gives them no policy vocabulary; 07 forbids exposing policy metadata via generic endpoints; 10 says only "schema discovery and generated fields; protected profiles ... limit metadata separately."
*Concrete failure:* (a) A caller whose field policy hides `credit_limit` calls the schema endpoint and learns the field exists, its type and constraints — the UI needs schema to render field checkboxes, so the endpoint will be reachable. C1 claims "A hidden field cannot become an inference channel"; schema is exactly such a channel for names/types. (b) Nothing in C1 authorizes DDL, so an actor with legacy schema capabilities can rename `tenant`, after which `tenant-access` matches nothing (fail-closed, so an availability incident) or add a field that lands outside every allow-list.
*Action:* Either define schema/DDL as an explicit administration action namespace (like policy CRUD) with its own grants and state that schema projection is filtered by the caller's field allow-lists, or state clearly in 02/08 that on ACL-enabled mounts schema endpoints are governed solely by pre-existing capability grants and that field *names* are not confidential in MVP. Add a T-case either way.

**O-13 — medium — First slice carries deferrable subsystems and a long serial critical path.**
*Evidence:* 12 dependency graph is effectively serial WP0→1→2→3→4b→5b→6→7b→8 across six repositories; C2 includes `sample` mode with its own selection/ordering/budget rules, `simulation` (hypothetical roles/groups/attributes), and structured expression disclosure with 16 KiB/256-node/256 KiB limits and `too_large` machinery.
*Assessment:* `sample` is user-directed (D08) so it is a choice, not a defect — but it is simultaneously the highest-leak-surface feature (T18 mixes requester∩subject readability, pre-write-ACL candidate selection and budget exhaustion) and the least load-bearing for the mission proofs. Hypothetical simulation and expression disclosure are similarly severable.
*Action (human trade-off):* Keep all three in C2 as capabilities advertised via `GET /layers`, but implement only plan + key-inspect + opaque restriction references in the first slice; land a single-owner (InGitDB-only) vertical through the daemon before three-owner composition, so O-01/O-03 are proven on the cheapest path.

**O-14 — low — Policy expressiveness gap may push authors to over-broad grants.**
*Evidence:* 03 comparisons `==, In, >, >=, <, <=`, no `not`; "Predicate-bearing deny rules are unsupported in this profile."
*Concrete failure:* "All rows except tenant `X`" and "everything that is not soft-deleted" are inexpressible without enumerating tenants/statuses; the natural workaround is an unconditional allow plus a hope that a lower layer restricts, which is exactly the pattern the architecture warns against.
*Action:* Document the inexpressible set and recommended patterns in 03; reserve `!=`/`NotIn` in the grammar now (rejected in v1 profile) so adding them later is not a version bump.

**O-15 — low — Policy-snapshot pin window is undocumented for operators.**
*Evidence:* 02 "DataTug pins its policy/principal snapshot through the outgoing request and response"; D15.
*Concrete failure:* Revocation latency equals the longest in-flight request, which spans two network hops plus a Git commit under an exclusive lock (`database.go:135-147`). Operators will assume revocation is immediate.
*Action:* State the guarantee as "revocation takes effect for operations admitted after the edit; in-flight operations may complete", bound it with an explicit request deadline, and put the number in 10.

**O-16 — low — `restrictions[].enforced` is undefined and misreadable.**
*Evidence:* 04 restriction shape `{..., enforced:boolean}` with no semantics anywhere.
*Action:* Define or rename, e.g. `enforcementState: enforced_here | delegated_downstream | pending_evidence`. A client that reads `enforced:false` as "this restriction did not apply" will display an unsafe result.

**O-17 — low — "13 mission proofs" are never enumerated in the packet.**
*Evidence:* 11 "Test assertions must verify all 13 mission proofs: T03(1,2,12), T04(3), T05/T06(4,5,6,13), T01/T02(7,8,9), T07(10,11)." The numbering covers 1–13 with no gaps, but the proofs themselves are not in the frozen packet.
*Action:* Enumerate them in 11 or drop the mapping; as written the pass gate is unverifiable by an independent party.

**O-18 — low — No concurrency/throughput criteria for the enforcement path.**
*Evidence:* `database.go:135-147` — one exclusive lock covering definition load, all record mutations, rollback and the Git ref update; 04 sets a 2 s dry-run budget; 08 routes `/evaluate` inspect and sample to bounded reads that take the shared lock.
*Concrete failure:* Concurrent Explain traffic plus one slow Git commit turns dry runs into `budget_exceeded` partial coverage, which the UI must present as "could not evaluate" — a confusing but correct fail-closed. No lock-acquisition timeout or mapping to `ACL_SOURCE_UNAVAILABLE` is specified.
*Action:* Define lock-wait timeout and its code mapping; add one data-path concurrency test alongside the existing two-client policy-revision race test in WP4b.

**O-19 — low — No version negotiation for the authorization contract.**
*Evidence:* 04 requires `apiVersion` be the exact string; `GET /layers` advertises "supported modes/formats" but not contract versions.
*Action:* Add `supportedApiVersions[]` to the layer descriptor and define client behavior on mismatch (fail closed with a distinct outer code, never degrade to a legacy unrestricted path — consistent with 02's rule on unrecognized versions).

**O-20 — low — Bootstrap principal and actor-vs-subject attribution.**
*Evidence:* `pkg/auth/middleware.go:53-57` constructs `&Principal{Owner:true}` with no ID; 06 "Owner tokens map to an explicit local bootstrap principal"; 04's only capability code is `ACL_CAPABILITY_DENIED` with `scope:principal`.
*Action:* Give the bootstrap principal a stable `(realm,kind,id)` so it can be bound in policies and audited; allow the blocker to say (safely, to self) whether the actor delegation or the subject policy failed — otherwise "why can't my app do this?" is undiagnosable.

---

## 3. Missing human decisions and disputed trade-offs

These are choices, not defects; each needs a human answer before C1–C6 freeze.

1. **Version namespace** (D03): `dtql.org/access/v1` vs. remaining on `dalgo.io/access/v*`. Owning the namespace is the right call for portability, but it commits to a dual-decoder path forever.
2. **Legacy precedence equality** (O-05): accept a documented tie-break change under the new apiVersion, or preserve unqualified rule names as the sort key. Not resolvable by test — it is a semantics choice.
3. **Existence-disclosure tier** (O-02): how much may `access:diagnostics` reveal? The packet's mission ("multiple blockers, Explain Access") pulls toward disclosure; the threat model pulls the other way. Choose explicitly; do not let the taxonomy decide by accident.
4. **"Public" by default** (D10): user-directed, and the packet correctly reinterprets it as *authenticated* discovery. I recommend renaming the enum values (`discoverable`/`restricted`, or `owner-visible`/`admin-only`) because operators will read `public` as "on the Internet" — a naming change now is far cheaper than a misconfiguration later.
5. **Policies inside the data repository** (O-06): a product decision with roadmap consequences for GitHub-backed InGitDB.
6. **Batch writes in or out of the ACL profile** (O-10).
7. **Remote conditional writes require readable evidence** (D13/C5): accepted limitation that write-only-with-condition principals cannot write remotely. Fine, but confirm nobody's intended fixture depends on it.
8. **First-slice size** (O-13): three owners at once vs. one-owner vertical first, and whether `sample`/`simulation`/expression disclosure ship in MVP.
9. **Schema/DDL governance** (O-12): are field *names* confidential in MVP or not? Answer once; several documents currently imply both.

---

## 4. Reuse and scope assessment

The reuse posture is the packet's strongest feature. It correctly refuses four expensive temptations: a new policy DSL, a second evaluator in Go/TypeScript, resurrecting the removed InGitDB HTTP server (ADR 0001), and a SQL-mutation authorization parser. It builds the wire contract around the *existing* DTQL condition shapes and the *existing* DALgo document kind, and it identifies concrete code to move rather than duplicate (DataTug's hidden-reference and alias checks → DALgo). The audit is credible where it is checkable: it flags the duplicate `openvaultdb-org/openvaultdb-go` clone as the same origin, notes the DALgo v0.64.4/v0.62.10 pin divergence, and distinguishes "docs say X, code does Y" in four repositories.

Scope risk is concentrated in three places: the DALgo `access` refactor from stop-at-first-failure to accumulate-independent-blockers (security-critical core, WP3, correctly rated high risk); the number of new surfaces landing simultaneously in C2; and the eight-stage serial critical path across six repositories with one-writer-per-worktree.

**Three things worth preserving verbatim:**

1. **The composition rule and its table in 02** — intersection only, no priority field, missing/failed/unknown participant never counts as allow, `disabled` reported as `disabled` rather than as an affirmative allow, and "enabled scope with zero policies → `ACL_CONFIGURATION_INVALID`, do not infer disabled from missing files." That last clause eliminates the most common real-world ACL failure mode.
2. **The dry-run purity split and the `allowed` definition** — plan reads no rows, inspect reads only supplied keys, sample reads bounded authorized candidates, `allowed:true` only for a complete final allow, `exhaustive` always false, and "Explain never authorizes a later operation." The recording-adapter tests in WP3 that *prove* purity are the right acceptance mechanism.
3. **The honest current-state audit (01) and its explicit non-claims** — "passing these is baseline evidence, not evidence that the proposed MVP works", "no frontend build or cross-repo E2E was run", the stale-docs list, and the duplicate-clone warning. Keep this discipline in every downstream document; it is what makes the rest of the packet reviewable.

---

## 5. Limits of this review and uncertainty

- I reviewed **only** the frozen packet plus the excerpts in `reviews/source-evidence.md`. I did not clone or read any repository at the SHAs in `audit-snapshots.json`, did not build anything, and did not run any test. I make **no** claim to have executed `go test`, Nx targets, or any E2E.
- Files that materially affect my conclusions but were **not** in the packet: `access/fields.go`, `access/session.go`, `access/document.go`, `access/principal.go` beyond line ~65, `condeval/*`, OpenVaultDB `pkg/core`, `pkg/mount`, `pkg/server/dtql.go`, all of `dalgo2openvaultdb`, and all Angular sources. Accordingly: **O-04** (field bounds under a terminal allow) is an inference from `policy.go:276-281` and may be resolved correctly in `fields.go`; **O-03(a)** depends on how the reference check will be wired at OpenVaultDB, which does not exist yet; parts of **O-01** depend on `SecureDB` nesting behavior I could not read. Treat these three as "verify against source before acting", not as established defects.
- Where I cite line numbers, they are the packet's excerpt line numbers, not proposed changes.
- I did not read the parallel review or any reconciliation, and I have made no attempt to anticipate or align with them. Overlap is coincidental.
- **Telemetry:** no model identifier, token counts, cache statistics, elapsed timings or USD cost were surfaced to me by tooling for this run. I am reporting them as **unavailable** rather than estimating; do not infer a zero, and do not substitute a subscription figure for API-reported cost.
- Review completion does not authorize implementation. Nothing in this document is a design approval; it is input to reconciliation and human sign-off.
