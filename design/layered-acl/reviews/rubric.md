# Independent review protocol and identical rubric

Review the frozen packet manifest and files 01–14, README and audit-snapshots. Reviewers receive the same source excerpts/snapshot references and user-direction log. Do not read the other review or reconciliation. Do not change the packet. Return findings, not implementation. Claims based on source should cite the packet document/section or exact repository path/symbol. Distinguish measured facts, code-inspection inferences and design preferences.

Reviewers: Astra B (independent frontier architecture review) and Opus. Capture exact executed model when returned by tooling. Never substitute another model and label it Opus. Run in parallel with the same frozen digest. Review completion does not authorize implementation.

## Assessment dimensions

Correctness; security; completeness; unnecessary complexity; reuse of existing architecture; DTQL/DALgo boundary; identity/actor/subject; layered enforcement; discovery/privacy/editability; structured blockers and completeness; plan/key/sample dry run and private/too-large conditions; DataTug UX; implementation feasibility; testability; MVP scope; future extensibility without scope creep.

Explicitly check the difficult cases: lower blockers after upper denial without mutation; row evidence confidentiality; sample not proving all records; private policies admin-only despite generic diagnostics grants; public-by-default user direction; remote read/write race and whole-row CAS; per-owner policy snapshot semantics; no-data-read plan; independent owner admin; conservative field alternatives; existing DALgo first-match hierarchy/legacy behavior; no separate InGitDB server.

## Required output

1. Verdict: ready, ready with changes, or not ready for implementation, with short rationale.
2. Findings table with stable IDs prefixed `A-` or `O-`: severity (`critical`, `high`, `medium`, `low`, `out-of-scope/future`), dimension, exact evidence, concrete failure/example, recommended action, and whether it blocks contract freeze. Avoid reporting the same root cause multiple times as independent value.
3. Missing human decisions and disputed trade-offs; say which are choices rather than bugs.
4. Reuse/scope assessment and up to three things explicitly worth preserving.
5. Limits of review and uncertainty. No claims to have run tests not actually run. No fabricated token/cost telemetry.

Severity: critical = plausible cross-boundary unauthorized access/data loss with no viable current contract; high = major security/correctness gap or implementation-blocking ambiguity; medium = meaningful reliability/interop/test gap with bounded fix; low = clarity/minor maintainability; future = outside current MVP, not a blocker merely for being absent.

## Separate reconciliation instructions

A separate agent (neither author nor reviewers) receives both sealed reviews and the frozen packet. De-duplicate and classify each root finding as consensus, complementary, conflicting, duplicate, incorrect/unsupported or optional. Assess severity/value and accept/reject/defer with rationale. Compare trade-offs for conflicts; do not invent a third architecture unless both options demonstrably fail. Flag human choices. Produce an actionable human report with accepted changes by work package and readiness verdict. The author may implement accepted **document** fixes, with a revision/change ledger, but no MVP code.

Meta-evaluation must identify each reviewer's unique useful findings, misses and assessable false positives; semantic overlap; whether Opus alone likely sufficed; incremental Astra value and future role-dependent workflow recommendation. This is one-project evidence, not universal model ranking. “Accepted” means reconciler recommendation until the human signs off.

## Cost/value metrics

For each reviewer and reconciler record exact model, request start/end and elapsed, input/output/cache tokens and total (with counting definition), actual reported USD or explicitly unavailable. Separate API-reported cost from an actual subscription invoice. Do not infer a missing zero or invent pricing. If telemetry isn't exposed, mark unavailable; optionally a labeled estimate with method, but avoid distracting pseudo-precision.

Overlap: after semantic deduplication let C = root findings independently raised by both; A/O = each review's root-finding count. Report Jaccard `C/(A+O−C)` and coverage overlap `C/min(A,O)` with numerator/denominator. Counts also include accepted findings, unique accepted, critical/high accepted. Cost per accepted = reported cost / accepted roots; per unique accepted similarly, undefined when zero or cost unavailable. Incremental Astra value = its unique accepted findings and severity relative to cost, plus useful independent corroboration separately. Do not count stylistic duplication as a second discovery. Capture limits of assessability and any runtime retry costs in the durable report.
