**Readiness: ready-with-changes for the A3 contract freeze.** The specified scoped-stage algorithm is coherent. Neither review establishes a new authorization bypass in a conforming implementation. Correct the canonical YAML mismatch and add the bounded clarifications/regression artifacts below; then perform focused correction verification. Implementation dispatch and native effects remain outside this review.

I read the frozen A3 packet, both sealed reviews and both run metadata files. I did not consult the live repository or web, edit files, or execute implementation tests.

| Reconciled root | Disposition | Severity | Freeze impact |
|---|---|---|---|
| R3-01: canonical quoting contradiction | Accept A3-01 | Medium | Actual canonical-artifact blocker |
| R3-02: missing discriminating regression artifacts | Accept A3-02 and bounded parts of O3-01/02/03/07 | Medium | Complete bounded declarative additions before this freeze; runtime execution belongs to implementation |
| R3-03: restored leaf/container explanation | Accept clarification component of O3-04 | Low | Small clarification and vector; no architectural decision |
| Alleged missing normalization, matcher extension or canonical merge requirements | Reject those substantive claims in O3-01/02/03 | — | Requirements already explicit |
| Mandatory caching, fixed schema ceilings, grammar duplication and definition rename | Reject as necessary fixes; defer optional engineering improvements | Future/low | No blocker |

**R3-01 — Correct the canonical YAML bytes.**

The README requires ambiguous scalars to use YAML double quotes and serializers to match the golden bytes. The revised `mask-policy.canonical.yaml` uses single quotes for `'*c*'`, `'*d'` and `'*'`. Both forms parse identically, but they cannot both satisfy that byte contract.

Change those scalars to double quotes while retaining the established canonical style. Check the complete golden against the stated formatting rules, its parsed document, and second-serialization idempotence when the prescribed canonical serializer is available. Semantic YAML equality alone does not resolve this defect. Do not claim a serializer test passed merely because two parsed documents agree.

This is Astra’s unique accepted substantive defect.

**R3-02 — Add focused examples that distinguish plausible implementation mistakes.**

The requirements exist, but the concrete corpus does not sufficiently discriminate several failures. Add a compact set of declarative vectors and canonical/transport goldens:

- **Adjacent excludes:** `include ["*"]; exclude ["a*"]; exclude ["b*"]; include ["*d"]`. Require `banana` denied and `bcd` restored. Include the canonical merged exclude OR-list. A raw-list evaluator can incorrectly admit `banana`; that demonstrates the value of the test, not permission for raw-list evaluation.
- **Changing canonical output:** provide adjacent includes and an unsorted, duplicated pattern list with exact merged/sorted/deduplicated output. This checks an existing canonicalization requirement.
- **Dotted repeated stars:** require `address.**` and `address.*` to agree for the concrete parent `address` and its descendants. The audited legacy parser’s terminal-`.*` handling makes this a useful integration regression beyond `**d` → `*d`.
- **Scoped fourth exclusion:** add `apple` remaining allowed under the existing fourth-stage `exclude ["a*"]` example. It never entered the prior exclusion/restoration subset, so the final exclusion cannot reach it.
- **Interior-star exclusion at protected operations:** `include ["*"]; exclude ["*secret*"]` must redact `my_secret_x` on wildcard read and deny an explicit reference or write to it. Preserve legacy grammar behavior in differential tests.
- **Complete restriction transport:** add a three-stage `field_mask` result, such as `address.*` → exclude `address.internal_*` → include `address.internal_public`, and require preservation through serialization and round-trip.
- **Callable assessment:** add an explicit three-stage stored-procedure example to the pure assessment/transport acceptance rows at applicable owners/hops. A matching gate still produces no native/procedure effects.

These are a single regression-completeness root, not seven new design defects. Both reviewers identify the adjacent-exclude gap. Astra uniquely contributes dotted-star parent equivalence, three-stage restriction transport and the fourth-stage unaffected-name case. Opus contributes useful explicit sort/dedup, interior-star read/write and callable-assessment cases.

**R3-03 — Explain restored leaves without introducing a new container veto.**

For:

```text
include ["*"]
exclude ["address"]
include ["address.city"]
```

the concrete `address.city` path is restored, while the concrete `address` path remains excluded. The existing text already requires evaluation on concrete paths, recursive projection, affected-descendant write checks and conservative whole-parent evidence. It also explicitly permits readable restored leaves.

Add an example making the consequence easy to implement: ordinary recursive projection can emit `address` as the structural container for authorized `city`; an otherwise-authorized update limited to `address.city` is not denied solely because the ancestor path’s own staged decision is excluded. Unauthorized siblings remain hidden/protected. Whole-parent evidence remains subject to complete-subtree authorization, and parent replacement/removal must validate all affected descendants, including removed or unchanged touched fields.

Do **not** adopt O3-04’s proposed blanket “denied container blocks whole-container read/replace/removal” sentence without these distinctions. It could suppress valid projected leaves or replace the already-selected affected-path rules with an additional ancestor veto. Include a read-projection and leaf-update vector, plus a parent replacement/removal case touching an excluded sibling.

This is Opus’s unique accepted clarification opportunity. It is not evidence that the architecture lacks field restoration semantics.

The remaining findings resolve as follows:

- **O3-01: reject high-severity fail-open/blocker characterization.** Section 19 explicitly says, “Before evaluation, merge each maximal consecutive run,” followed by “Only then apply scoped stages.” The README repeats it. The counterexample violates that precondition. Adding “over the normalized, merged stage list” immediately before the streaming explanation is useful local clarity; its regression belongs to R3-02.
- **O3-02: reject the missing-matcher-design characterization.** Section 19 requires arbitrary-position anchored stars and bounded glob matching while preserving the legacy parser/matcher. Routing new patterns through an unchanged matcher that cannot implement the grammar is already nonconforming. A compiled matcher variant is a possible implementation, not a contract requirement. Accept the regression and, optionally, a short statement that legacy matching cannot substitute for new-profile semantics.
- **O3-03: reject the claimed unspecified canonical representation.** Section 19 describes canonical serialization within each **merged stage**, preserves action-change order and identifies one normalized AST for JSON/YAML; the README also mandates canonical merging. Preserving semantic stage order does not require preserving redundant raw same-action boundaries. An explicit “canonical output emits merged stages” sentence and goldens improve clarity without changing the contract.
- **O3-05: reject “unbounded request” and mandatory caching/schema-cap recommendations.** Finite advertised raw-policy limits, whole-body limits, operational work/depth budgets, deadlines and explicit failure behavior are already required. Worst-case repeated work can motivate profiling; the arithmetic does not establish an observed availability defect. Per-document aggregate limits are not weaker than equivalent per-mask limits, and fixed reference `maxItems` values would conflict with configurable operational limits and variable-length syntax. Memoization may be explored during implementation within correct policy/snapshot boundaries; it is not required for semantic safety.
- **O3-06: reject missing validation/security characterization.** The README explicitly requires semantic validators to enforce field grammar in addition to JSON Schema. A structural validator accepting a string does not make it an accepted policy. Additional structural regex coverage could improve early feedback, but cannot replace semantic validation and is not a freeze blocker. Preserve the existing whitespace-rejection case; malformed dotted-segment cases may be added cheaply.
- **O3-07: accept the concrete regression additions under R3-02/R3-03**, with no separate root or severity multiplication. Broad staged behavior at owners/hops is already mandated.
- **O3-08: defer optional naming cleanup.** Reusing a structural `collectionMask` definition for callable-name masks does not prove incorrect generated types or authorization behavior. A neutral shared name may help maintainability, but a schema rename is unnecessary to close this review.

No new human semantic decision is necessary: scoped restoration and nonrecursive `**` are already confirmed. After bounded document/artifact correction and verification, the separate implementation sign-off remains the applicable authorization boundary.

The two-model review added useful coverage on this project, but its value should be assessed by accepted roots rather than finding count: one shared regression root, Astra’s unique canonical defect, and Opus’s unique container clarification, with complementary test cases. Astra’s report was more accurately calibrated to explicit requirements; several Opus findings recast already-mandated behavior as missing design. This single review does not support a general model-quality or cost-effectiveness ranking.

| Exposed metadata | Astra | Opus run |
|---|---:|---:|
| Reported elapsed seconds | 183.932 | 190.451 |
| Input tokens | 454,980 | 2 uncached direct |
| Cached input/read tokens | 412,032, included in input | 0 |
| Cache creation/write tokens | 0 | 64,308 |
| Output tokens | 3,702 | 13,702 |
| Reasoning/thinking tokens, included in output | 1,849 | 9,356 |
| Reported total tokens | 458,682 | No top-level total reported |
| Reported cost USD | Unavailable | 1.030099 |

Opus metadata separately reports `claude-opus-5` at **$0.9856400000000001** and auxiliary `claude-haiku-4-5-20251001` at **$0.044459**; the latter exposes 44,379 input and 16 output tokens. These model-usage records must not be added again to the top-level Opus usage as if disjoint. Astra’s cumulative input includes repeated cached context, so its input number is not directly comparable to Opus’s uncached-input field. Astra cost is unknown, not zero; combined USD and this reconciler’s USD are unavailable.
