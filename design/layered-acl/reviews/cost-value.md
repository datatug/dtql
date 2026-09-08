# Review cost/value evidence — one project

Source of findings/dispositions/counts: [separate reconciliation](reconciliation.md). Telemetry: [Astra](astra-run-metadata.json), [Opus](opus-run-metadata.json), [normalized Opus categories](opus-normalized-usage.json), [reconciler](reconciler-run-metadata.json). No cost rates were invented. Reported list-price usage cost is not a verified subscription invoice.

| Run | Exact model | Input including cache once | Output including reasoning | Total | Elapsed | Reported USD |
|---|---|---:|---:|---:|---:|---:|
| Astra B | gpt-6-astra | 461,622 | 8,132 | 469,754 | 433.064 s | Unavailable |
| Opus main inference | claude-opus-5 | 63,258 | 40,869 | 104,127 | Included in CLI elapsed | $1.654295 |
| Opus CLI ancillary call | claude-haiku-4-5-20251001 | 44,474 | 16 | 44,490 | Included in CLI elapsed | $0.044554 |
| **Complete Opus CLI run** | Main + ancillary above | **107,732** | **40,885** | **148,617** | **585.629 s** | **$1.698849** |
| Separate reconciler | gpt-6-astra | 777,121 | 7,571 | 784,692 | 325.551 s | Unavailable |

Astra cumulative input includes **403,584** cached-read tokens; reconciler cumulative input includes **691,200**. These totals include repeated context across tool/model turns and are not unique input sizes. Opus main input includes **63,256** one-hour cache-creation tokens and **2** other input tokens; cache creation is not a cache hit. Reasoning/thinking already included in output: Astra **4,879**, Opus **28,416**, reconciler **999**. Do not add them again.

The only known monetary component is the complete Opus CLI reported cost. Total review-plus-reconciliation dollar cost and Astra/reconciler cost per finding are **unavailable**, not zero. Completed run metadata exposes no separately billable retries; the auxiliary Haiku call is included explicitly. Tokenization, caching, harness overhead and repeated source reads differ, so raw token totals are not a model-efficiency ranking.

## Findings and overlap

Counts below are reconciler recommendations awaiting human sign-off. Twenty-six submitted IDs map to 24 semantic roots; 17 roots are accepted, five high, none critical. O-01/O-09 internally overlap, and A-06 independently shares that evidence root.

| Metric | Astra B | Opus |
|---|---:|---:|
| Submitted IDs | 6 | 20 |
| Internally deduplicated roots | 6 | 19 |
| Accepted roots | 6 | 12 |
| Unique accepted roots | 5 | 11 |
| Accepted high roots | 4 | 2 |
| Unique accepted high roots | 3 | 1 |
| Cost per accepted root | Unavailable | $0.1416 |
| Cost per unique accepted root | Unavailable | $0.1544 |

Overlap: one shared root. Root Jaccard **1/(6+19−1)=4.2%**; smaller-review coverage **1/6=16.7%**; accepted-root Jaccard **1/(6+12−1)=5.9%**. Thematic overlap is higher: discussing “identity” or “batches” is not necessarily discovering the same defect. See reconciliation's full root mapping for the counting rule.

## Value assessment

Astra added five unique accepted roots, including three high: typed binding collisions, repeated-key batch candidate semantics and incomplete sampled-result schema. Its additional bounds/crash-recovery findings and concrete hidden-versus-absent evidence counterexample were useful. No false-positive root was established among its six, but it missed several disclosure/transport concerns.

Opus added eleven unique accepted roots, including protected-row diagnostic leakage, visibility replacement behavior, omission authority, batch transport, adapter semantic tests and schema boundaries. Its breadth required source-backed triage: O-05 was false, O-04's suspected source defect was unsupported, and suggested changes to remote field exemptions or user-requested MVP modes were not accepted. Do not conflate a rejected preference with a factual false positive; a universal false-positive rate is not assessable from this review alone.

Opus alone likely would have blocked premature contract freeze, but it did not identify Astra's three unique accepted high roots. Astra therefore added material security/correctness value. Whether that value justified its monetary cost cannot be quantified because its cost was not exposed.

For similar cross-repository authorization contracts, the reconciler recommends complementary broad architecture/disclosure and adversarial contract reviewers, plus source-backed reconciliation. For future comparisons, use consistent input delivery and complete evidence excerpts, capture actual cost telemetry at dispatch, and accumulate results across projects. This is one project, not a general ranking of models.
