# A3 human checkpoint — scoped mask stages

**Ready for human sign-off; no implementation has started.** The user-confirmed semantics now appear in the design, schemas, canonical examples, UI/work packages and acceptance plan. A later include is confined to the set selected by its preceding exclusion and ancestors. It cannot override a separate mandatory policy or layer.

| Material | Link |
|---|---|
| Exact staged semantics, wildcard grammar, bounds | [19 — scoped stages](../../19-scoped-mask-stages.md) |
| Collection/column evidence and enforcement | [18 — integration](../../18-mask-supplement.md) |
| Execution classes and procedure policy | [03 — policy contract](../../03-policy-format.md) |
| Executable plan and E2E | [16 — package cards](../../16-handoff.md), [11 — acceptance](../../11-acceptance.md) |
| Schemas and examples | [Contract artifacts](../../contracts/README.md) |
| Independent reviews | [Astra](astra-review.md), [Opus](opus-review.md) |
| Separate reconciliation and verification | [Reconciliation](reconciler-review.md), [correction verification](reconciler-verification-review.md), [applied fixes](correction-ledger.md) |
| Review cost/value | [Report](cost-value.md), [actual metrics](metrics.json) |

Both independent reviews found the scoped model coherent and returned ready-with-changes. The reconciler accepted three roots: canonical quoting, regression coverage and container/leaf explanation. Corrections are applied. Separate verification found **no remaining concrete A3 freeze blockers**, and verified the final input SHA and all 48 manifest entries.

The updated corpus covers adjacent include/exclude grouping, interior stars, `**d` normalization, `address.**` parent behavior, additional scoped alternation, three-stage transport, and restored leaf access without an ancestor veto or excluded-sibling overwrite. Thirty structural fixtures validate. Four canonical YAML artifacts match prescribed bytes and pass a local document harness round-trip/idempotence check. Production DALgo serializer, ACL and real browser/storage tests remain implementation obligations.

No new semantic decision remains open. The existing protected-profile limits remain: real DTQL read and typed UPDATE; native/procedure effects execution unsupported even after mask admission; finite advertised operational budgets; independent owner enforcement; private evidence and policy visibility protections. Variable-length stages are preserved and never silently truncated.

Astra and Opus each contributed two accepted roots, one shared and one unique. Opus reported $1.030099 including its exposed helper; Astra/reconciler costs are unavailable. See the report for interpretation and measurement limits.

[Frozen review input](input-A3.txt) and [manifest](manifest-A3.json) preserve what both reviewers saw. [Corrected input](input-final.txt) and [final manifest](manifest-final.json) preserve the verified revision. Previous A2/S1 review evidence remains unchanged in round-2; its flat-mask readiness is historical.

The original workflow requires this checkpoint before Corpus dispatch. The next human sign-off can freeze the reviewed contracts and authorize implementation handoff. This task has changed documentation/specification artifacts only; no implementation agents were dispatched.
