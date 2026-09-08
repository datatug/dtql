# A3 scoped-mask delta review (independent)

**Verdict: ready-with-changes.** The scoped stage model is coherent, matches the user's example, and the schema/fixtures encode the key structural invariants (first stage include, no empty stages, no mixed-key stage, flat form rejected). Four issues should be closed in the contract text/artifacts before the new freeze; the rest are bounded follow-ups.

---

## O3-01 — Streaming algorithm is unsafe on un-normalized stage lists (fail-open)
**Severity: high. Freeze blocker: yes (text-only fix).**
Evidence: 19 §Selected-set semantics: "process the stages from the beginning: if it matches a stage, set its inclusion state to that stage's action and continue; at the first nonmatching stage, stop." That paragraph is stated over *stages*, while the merge precondition lives elsewhere (19 §Exact integration: "Before evaluation, merge each maximal consecutive run"; 18 §Post-review; README). Adjacent same-action stages are explicitly *valid input* (fixtures.json → adjacent-include-stages.json valid).
Counterexample: `stages:[{include:["*"]},{exclude:["a*"]},{exclude:["b*"]}]`, name `b1`. Literal streaming: stage1 matches → include; stage2 `a*` does not match → stop → **included**. Merged evaluation: exclude run `{a*,b*}` matches → **excluded**. Two conforming implementations disagree, and the divergence direction is *admit a name the author excluded*. (The published `banana` vector only exhibits the harmless over-restrictive direction, so it does not catch this.)
Action: in the same paragraph, state that the algorithm is defined only over the canonically merged stage list, and that evaluating raw runs is nonconforming. Add one vector: `include *`, `exclude a*`, `exclude b*` → `b1` denied.

## O3-02 — Reuse seam: legacy `segmentMatches` silently mis-evaluates interior/multi-star patterns
**Severity: medium-high. Freeze blocker: no (needs one normative sentence + vector).**
Evidence: audited `access/fields.go` lines 52 (parser rejects >1 star / interior star), 63–74 (`segmentMatches` handles only `*`, `prefix*`, `*suffix`). 19 says "Extend DALgo's parsed mask machinery… preserving its legacy fields parser/matcher" and "bounded glob matching", but does not forbid routing new-profile patterns through the existing matcher, and both would live in the same `fieldPattern`/`fieldSet` structures.
Counterexample: `fieldMask: include ["*"], exclude ["*secret*"]`. Through `segmentMatches`, `HasPrefix(pattern,"*")` → `HasSuffix(segment,"secret*")`, which is false for `my_secret_x` → the exclusion matches nothing → **`my_secret_x` is disclosed**. An exclusion evaluated by the wrong matcher fails open, unlike an inclusion.
Action: require that new-profile patterns carry an explicit compiled matcher variant (multi-star capable), that `segmentMatches`/`parseFieldPatterns` are never applied to them, and add a negative unit vector asserting `*secret*` excludes `my_secret_x` and that a legacy-parsed pattern set cannot contain interior stars. Also note `parseFieldPatterns` line 39 `TrimSpace` must not be reused, since new masks must *reject* whitespace (19; semantic-vectors `new-mask-whitespace`).

## O3-03 — Canonical form of adjacent same-action stages is unspecified in the persisted document
**Severity: medium. Freeze blocker: yes (canonical bytes are frozen artifacts).**
Evidence: 19 requires "Stage order is semantic and MUST be preserved through editing/serialization" and canonicalization that "never sorts stages"; 18/README require "Merge adjacent same-action stages… preserving action-change order". Whether merging changes the *stored/canonical* document or only the pre-evaluation AST is not stated. `adjacent-include-stages.json` is a valid fixture with **no canonical golden**.
Counterexample: a user saves `[{include:[a*]},{include:[b*]},{exclude:[*c*]}]`. Owner A canonicalizes to two stages, owner B preserves three. Canonical bytes, policy content digest / ETag and the byte-identical-idempotence promise (03 §round-trip) differ per owner; the UI diff shows a phantom change, and T21's "persist/reload/canonicalize without reordering stages" is untestable.
Action: state explicitly that canonical serialization emits the merged run form (recommended, matching the evaluation AST), and add `adjacent-include-stages.canonical.yaml`/`.json` golden plus one dedupe/sort-reordering golden (e.g. input `exclude ["secret_*","address.internal_*","secret_*"]`), which no current fixture exercises since all shipped pattern lists are already sorted.

## O3-04 — Container (non-leaf) field paths: decision semantics undefined under restoration
**Severity: medium. Freeze blocker: no.**
Evidence: 19 says decisions are "evaluated on the concrete field path under all prior selections" and "A restored child does not imply that the whole parent is readable", but never says whether an intermediate container path is itself masked. Existing DALgo evaluates leaves only (`disallowedPaths` walk, lines 189–212; `redactMap` 231+).
Counterexample: `include ["*"], exclude ["address"], include ["address.city"]`. Path `address.city` → include (stage3). Path `address` → exclude (stage3 has 2 segments, cannot match a 1-segment path, so streaming stops at the exclude). Is `UPDATE address.city` allowed (leaf-only decisions) or denied (container denied)? Is `address` emitted as a container holding only `city`? Both readings are defensible and produce different read output and different write outcomes.
Action: one normative sentence: mask decisions apply to leaf paths and to explicitly referenced paths; a denied container blocks whole-container read/replace/removal and whole-container evidence, but does not by itself deny an authorized restored leaf beneath it. Add both vectors (read projection and `set address.city`).

## O3-05 — Mask work is bounded per document, not per request
**Severity: medium. Freeze blocker: no.**
Evidence: 19 advertises `maxMaskStages=1024`, `maxMaskPatterns=4096` **per policy document**, `maxMaskPatternBytes=128`; schema `collectionMask/fieldMask.stages` has `minItems:1` and no `maxItems`, and pattern arrays are unbounded. C5 bounds rows (1000), columns (32) and the 10 s execution deadline, but nothing bounds mask evaluations per request.
Counterexample: an authorized policy editor publishes a fieldMask at the advertised budget; a legal query returns 1000 rows × 32 columns and each field path is re-evaluated against ~4096 patterns → ~1.3×10⁸ glob matches inside the execution deadline; intersecting three mandatory owners triples it. Result is deadline exhaustion / indeterminate on ordinary traffic — a self-inflicted availability cliff, not a bypass.
Action (bounded): state that mask decisions are memoized per distinct field path (and per collection name) within one assessment, so cost is O(distinct paths × patterns) not O(rows × …); and either lower the reference per-mask pattern budget or advertise it per mask rather than per document. Add `maxItems` to the schema arrays at the reference limit so structural validation cannot be the only gate for a JSON `policyWrite` document (whose object variant has no size bound, unlike the 1 MiB string variant).

## O3-06 — `fieldPattern` has no grammar in the schema, unlike `nameMask`
**Severity: medium. Freeze blocker: no.**
Evidence: schema `$defs.nameMask` carries `pattern:"^[^./\\\\\\x00-\\x20\\x7f?\\[\\]]+$"`; `$defs.fieldPattern` is only `minLength:1,maxLength:128`. 19 requires new masks to reject whitespace, controls, escapes, `?` and character classes — semantically only.
Counterexample: `fieldMask: exclude ["addr?ss.secret", " secret_*", "a..b"]` passes JSON Schema. A consumer generating validators from the pinned artifact (README: "Implementers generate/check DTOs against the pinned artifact") accepts it; a lenient implementation reusing legacy `TrimSpace` turns `" secret_*"` into an active exclusion while a strict owner rejects the document — divergent effective policy across owners for the same bytes.
Action: give `fieldPattern` a regex permitting dots as segment separators and forbidding empty segments, whitespace, controls, `\`, `?`, `[`, `]` (e.g. dot-joined runs of the `nameMask` character class), keeping semantic validation as defence in depth.

## O3-07 — Delta test corpus gaps
**Severity: low-medium. Freeze blocker: no.**
Evidence: 11 §A3 acceptance and T21 cover the six named rows, adjacent *include* grouping, a fourth exclude, cross-owner non-bypass, `**d` canonicalization, budget limit/limit+1 and parent-evidence present/absent. Not covered: (a) adjacent **exclude** run (O3-01's fail-open direction); (b) an exclusion whose pattern has an interior star actually removing a column at read and write (O3-02); (c) pattern sort/dedup that changes order (O3-03); (d) container-path semantics (O3-04); (e) a ≥3-stage **stored_procedure** mask at each hop — T20 only exercises two stages, while 03's example ships three.
Action: add these five vectors/rows; they reuse existing fixtures and add no new harness.

## O3-08 — Naming: procedure mask reuses `$defs/collectionMask`
**Severity: low. Freeze blocker: no.** `executionGate.allow[stored_procedure].mask` `$ref`s `collectionMask`, so generated Go/TS DTOs will type a callable-name mask as `CollectionMask`. Counterexample: a WP1 TS consumer validating a procedure mask against a "collection" type, or a reviewer assuming collection-gate semantics apply. Action: rename the shared definition (e.g. `nameStagesMask`) and alias `collectionMask` to it; pure artifact rename, do before freeze with the other schema edits.

---

## Positive confirmations (no action)
- The "stop at first nonmatching stage" rule is provably equivalent to the recursive scoped-set definition **on merged lists**: membership in Sₖ requires matching stages 1..k, so the last applicable state is the action of the stage before the first mismatch. The published table (apple/bacon/abcd/acf/bdf/zcd) and `scoped-mask-vectors.json` are correct under it, including `zcd` (never enters S₁).
- Structural invariants are correctly expressed via `prefixItems`+`items`: first stage must be include; `invalid-first-exclude`, `invalid-empty-stages`, `invalid-mixed-stage`, `invalid-flat-mask` all reject for the stated reasons.
- `fields`/`fieldMask` exclusivity and deny-rule prohibition are enforced in-schema (`not:{required:["fields","fieldMask"]}`, deny branch), matching `invalid-fields-and-mask`.
- Cross-policy intersection, "no stage restores another owner's denial", conservative parent evidence (including the child-absent case), disabled enumerable fast path for exclusion-bearing masks, `**d`≡`*d` with no path recursion, and opaque-reference disclosure ("never return only the includes") are explicit and adequate; I do not count them as gaps.

## Limitations
Review is confined to the supplied delta packet. I could not consult the live repositories: the DALgo excerpt ends mid-`redactMap` (line 240), so I could not verify `allows`/redaction call sites, the write/affected-path code, or the C7 gate implementation surface. No runtime behaviour, performance measurement, telemetry or usage/cost data exists or is invented here; O3-05's arithmetic is derived only from the advertised limits in 19 and C5. Fixture JSON was reviewed as supplied (compacted), not re-validated against a validator. I have not seen the peer reviewer's findings, and I have made no edits and proposed no implementation.
