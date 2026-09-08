# Executable implementation plan and handoff

Status: A2 plan incorporating approved A1 recommendations. [Executable package cards](16-handoff.md) refine the original WP numbers below; C7 adds route/mask controls. Design work packages only. Do not start implementation or dispatch Corpus until human approval after independent review/reconciliation. Estimates below are relative effort/risk judgments, not measured hours or token costs. Re-audit pinned dependencies at implementation start.

## Shared contract freeze

| Contract | Freeze material | Human gate |
|---|---|---|
| C1 | Policy version/grammar, hierarchy, fields, privacy classification, legacy conversion (03) | Approve version namespace and preserve DALgo semantics. |
| C2 | Operation descriptors, result/coverage/taxonomy, discover/CRUD/evaluate, plan/inspect/sample, omission rules (04) | Approve limits and diagnostic authority. |
| C3 | Canonical realm/subject/actor, trusted context, private-policy admin, deployment identity profile (06/10) | Confirm hosted ADR reuse and self-hosted fixture profile. |
| C4 | Provider/reader/manager/inspector contracts, capabilities, pure evaluation, errors (05) | Approve additive DALgo implementation home. |
| C5 | Whole-record revision evidence, conditional write preconditions, short-operation policy snapshots (02/08) | Approve consistency limitations and remote evidence requirement. |
| C7 | Execution-class gate and canonical procedure include/exclude masks (03/04); unsupported native effects remain closed | User-requested controls; exact A2 forms reviewed with plan. |
| C6 | Structured query subset, key-targeted set-assignment UPDATE, final-image constraints and sample bounds | Approve a narrow real vertical slice and explicit unsupported behavior. |

Freeze a reviewed contract revision and machine-readable golden vectors in DTQL; every downstream package references that exact revision. Later contract changes require coordinated review, not divergent implementation assumptions. No parser module extraction, new InGitDB HTTP server, custom ACL service or long-running transaction subsystem in these packages.

## Dependency graph and order

```text
Human sign-off → WP0 baseline/pins → WP1 contract fixtures
WP1 → WP2 DALgo model/evaluator → WP3 DALgo enforcement
WP1 → WP4a InGitDB policy storage scaffolding (completion also requires WP2a/b)
WP1 → WP5a OpenVaultDB transport/store scaffolding (completion also requires WP2a/b)
WP1 → WP7a DataTug forms against shared contract fixtures
WP3 + WP4a → WP4b InGitDB secured adapter/revisions
WP3 + WP4b + WP5a → WP5b OpenVaultDB enforcement + WP5c HTTP driver (paired)
WP3 + WP5b + WP5c → WP6 DataTug daemon integration
WP6 + WP7a → WP7b live DataTug UX
WP4b + WP5b + WP6 + WP7b → WP8 vertical E2E/security → WP9 release/docs
```

Parallel work uses fixed contract fixtures and isolated worktrees. One writer per worktree. Avoid splitting `access` internals across competing branches until WP2 API stabilizes. Provider-first version releases, then one downstream bump per verified wave; no repeated uncoordinated library bumps. An implementation agent receiving a package must have the contract revision, affected files, dependency versions and tests below.

## WP0 — Baselines, contracts and release topology

Repos: all audited providers/consumers. Dependencies: human sign-off. Tasks: inspect current SHAs/AGENTS, identify current module versions and current NX targets without installing arbitrary latest tooling; inventory transitive adapters (especially `dalgo2openvaultdb`, InGitDB nested module); record compatibility matrix; confirm existing local/hosted auth integration owner; copy accepted source corrections into owner-doc update list. Run focused legacy suites once. Acceptance: reproducible pinned baseline and exact commands for subsequent package checks; stale duplicate OpenVaultDB clone excluded. Risk/effort: low-medium; module drift could invalidate API sketches. Do not broadly synchronize or change unrelated repositories.

## WP1 — DTQL contract schemas and conformance fixtures

Repos: `datatug/dtql` (normative spec), DALgo test consumer, DataTug TS fixture consumer. Dependencies: WP0 and C1–C6 decisions. Tasks: consume the completed A2 schemas and golden specification artifacts; implement validation/distribution and expand the semantic fixture harness; canonical YAML expected bytes; compatibility vectors for legacy policy conversion; schema freshness/validation check. Preserve existing query schema unchanged. Acceptance: all fixture examples validate; duplicate-key/depth checks specified outside JSON Schema where necessary; semantic validators reject target/query mismatch and private metadata leaks; no prose/schema discrepancy. Tests: T04/T09/T15/T18/T19 fixtures and code-taxonomy exhaustiveness. Risk/effort: medium; avoid overengineering a generic schema generation framework.

## WP2 — DALgo documents, principal snapshots and multi-decisions

Repo: `dal-go/dalgo`, primarily `access/{document,principal,policy,context}.go` and DTO converter package. Depends WP1. Tasks: C1 strict decoder/serializer and editable AST; convert/compile into existing rules; structured binding/rule attribution; immutable verified principal adapter; add complete assessment/obligation types and pure policy adapter; preserve legacy `Policy`, `Decision`, `DeniedError` compatibility; bound caches. Acceptance: semantic/canonical round-trip, legacy unchanged decisions, structured multi-policy blockers and final/conditional distinction; no English parsing. Tests: focused access/document/principal/condition tests, property policy-order monotonicity, strict input fuzz corpus, import-boundary test. Risk/effort: medium-high due to residual semantics; review this before WP3.

## WP3 — DALgo enforcement and shared dry run

Repo: `dal-go/dalgo`, `access/{session,condition,fields,write}.go`, `condeval`, existing end2end access harness. Depends WP2. Tasks: shared plan/evidence assessment; collect independent blockers across policies/targets; query reference checks migrated from DataTug; before-paging row filters, conservative field bounds; pre/post image full touched-field validation including set removal/nested set; capability checks; no-data-read plan and bounded inspect; pure provider/custom handling. Acceptance: T01/T04/T07/T08/T10/T16 unit/recording-adapter behaviors; zero side effects on denial/dry run; existing tests pass. Tests: focused access+dtql+condeval, recording adapters, meaningful property/fuzz cases; one full provider suite before release. Risk/effort: high, core security critical. No new arbitrary SQL analyzer.

## WP4a/b — InGitDB owner storage and enforcement

Repos: `ingitdb/ingitdb-go` root config/schema, `ingitdb/dalgo2ingitdb`, relevant schema docs in `ingitdb-specs`. WP4a depends WP1; WP4b depends WP3+WP4a. Tasks: explicit enabled manifest, strict path-safe policy loader, owner admin provider/CRUD, revisions/CAS and Git persistence; integrate public constructor/transactions with secured facade; private enforcement evidence and owner-issued whole-record revisions; maintain schema capability forwarding; atomic revision checks under existing write lock. Acceptance: direct adapter and OVDB-mount semantics identical; policy reload and failed commit rollback; no policy-as-record exposure; T03/T05/T14/T16. Tests: existing CRUD/conformance/file-lock tests plus policy integrity and two-client revision-race tests. Risk/effort: high; nested module releases and wrapper interface loss. No server, cloud GitHub ACL write support or new locking architecture.

## WP5a/b — OpenVaultDB and HTTP driver

Repos: `openvaultdb/openvaultdb-go`, `openvaultdb/openvaultdb` docs, `dal-go/dalgo2openvaultdb`, optional `openvaultdb/ovdb` bootstrap config flags only. WP5a depends WP1; WP5b depends WP3+WP4b+WP5a. Tasks: own policy store; source registry and capability discovery; C2 routes and redaction; stable subject/actor mapping onto existing token store; control-plane admin grants, private policy visibility; collection-aware DTQL capability check; secured core/mount all paths; revision-bearing point read and atomic write precondition; typed driver result/error transport; bounds and sample endpoint. Acceptance: real HTTP three-blocker response, no owner-token override, CRUD revision/persistence, native source opacity and SQLite smoke, no writes in evaluate. Tests: auth/server/core/mount/driver tests, T07–T19 applicable cases, compatibility tests for old non-ACL clients. Risk/effort: high; existing error mapping and broad auth shortcuts must be audited route-by-route.

## WP6 — DataTug daemon, provider reuse and secured connection

Repo: `datatug/datatug-cli`. Depends WP3+WP5b. Tasks: reuse accesspolicies loader/run while removing duplicate enforcement and English attribution parsing; authenticated connection-scoped provider routes; own manifest persistence; principal context preservation; lower inspection despite upper static denial; remote conditional evidence and revision precondition; structured key UPDATE endpoint using existing connection path, leaving read-only builder invariant; safe sampling/condition return. Acceptance: same evaluator supports real operation and Explain, HTTP caller cannot use local no-policies/principal flags, stale evidence fails, T01–T08 and T11–T13. Tests: focused accesspolicies+server/API tests; trace no raw connection fallback. Risk/effort: medium-high; legacy execution path and newer spec not necessarily identical implementations.

## WP7a/b — DataTug browser policy workspace and record edit

Repo: `datatug/datatug-apps`. WP7a depends WP1, WP7b depends WP6+WP7a. Tasks: owner/layer list, policy form preserving full supported AST, read-only custom/private projection, admin capability checks as UI hints, ETag save/conflict/reload; Explain plan/IDs/top-N selectors and principal/simulation controls; structured blockers and private/too-large restrictions; stale revision markers; simple key-targeted record-edit path. Acceptance: all visible UI states match C2; no browser evaluator, owner credentials or simulated execution; edits persist at owner; inaccessible policies absent/generic. Tests: NX-discovered unit/component targets, keyboard/status accessibility, HTTP contract fixtures, T05/T06/T09/T18/T19. Risk/effort: medium; full native query-builder redesign is not needed.

## WP8 — Real vertical acceptance and security verification

Repos: DataTug E2E harness plus existing OpenVaultDB/InGitDB conformance harnesses. Depends WP4b/5b/6/7b. Tasks: provision fixture tokens/current membership/policies, real services and Git data, execute T01–T21, capture sanitized artifacts; verify policy edit→restart/reconnect→conditions/inspect/sample→query→UPDATE; simulate stale data/policy and unavailable providers; separate direct-lower routes. Acceptance: all mandatory tests pass with actual storage and no mocked lower decisions, no skipped critical cases; meaningful evidence of no denied writes. Tests: one complete vertical run plus focused reruns on actual changes/failures, not repeated fleet scans. Risk/effort: medium-high; E2E harness integration is enabling work, not optional polish.

## WP9 — Documentation, release and approved handoff completion

Repos: all owner docs and module consumers. Depends WP8. Tasks: update DALgo stale access docs/spec status, OpenVaultDB stale auth/DTQL docs, InGitDB root config docs/server direction, DataTug feature specs; publish DTQL schemas under approved namespace; pin provider releases in dependency order; retain release matrix and migration examples. Conformance claim is version/profile-specific. Acceptance: package versions and supported capabilities documented, legacy disabled behavior tested, no canonical spec claims that contradict code. Use repository-required specification tooling where available; do not silently rewrite unrelated existing decisions. Risk/effort: low-medium.

## Scope ledger

MVP: owner policy persistence/discovery/read/edit; users/roles/groups; public default; all-layer query and key UPDATE; structured blockers; plan/IDs/bounded sample; opaque condition references; custom read-only pure policy; actual E2E. Enabling: whole-row revision contract, DALgo hidden-reference checks, typed HTTP errors, stable actor/subject resolver, tests/schema fixtures. Follow-up: richer forms/raw editor, more write operators/bulk mutation, cloud adapter policy editing, policy import tooling, native ACL import. Future: custom ACL service and logical change sets. Out of scope: standalone InGitDB server, new auth protocol, arbitrary SQL authorization parser, distributed/global transactions, approval workflow engine, cross-source atomic mutation, billing and marketing implementation.
