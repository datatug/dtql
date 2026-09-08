# Review cost/value — implementation plan and mask supplement

Use [metrics.json](metrics.json) and original run metadata for exact counts. [Independent reconciliation](reconciler-review.md) explains root classification and incremental findings. This is one project's evidence, not a universal ranking.

| Measure, main + supplement | Astra | Opus |
|---|---:|---:|
| Accepted roots | 6 | 17 |
| Unique accepted roots | 4 | 15 |
| Independently established high-severity contributions | 2 | 0 |
| Tooling-reported USD | Unavailable | $3.215762 |
| Cost per accepted root | Unavailable | ~$0.189 |
| Cost per unique accepted root | Unavailable | ~$0.214 |

There are 21 accepted roots overall, including low document/fixture clarifications. Two shared roots yield Jaccard 2/35=5.7% and smaller-review overlap 2/6=33.3%. Both reviews contribute to those roots, but Astra identified the deeper dynamic-worker and pre-selection-denial failures; Opus identified the narrower interface/ID issues. Do not treat accepted-root volume as equal severity or value.

Astra also uniquely identified whole-set revision leakage, package completion dependencies, and incomplete parent-object evidence. Opus supplied useful parser, batch, evidence-list, digest-reuse and test-ownership checks. Several Opus high claims overlooked explicit trusted-process, literal-dot rejection, mandatory-policy intersection or mask-enforcement requirements. The reconciler rejected or narrowed those findings. Astra missed useful bounded interoperability details.

For this plan, both contributed value. Opus alone would likely have missed freeze-blocking contract issues; Astra alone would have missed useful checklist coverage. Use an Astra reviewer for security and executable-contract boundaries, Opus for broad interface/test coverage, and one separate reconciler. Monetary ROI for the second reviewer cannot be calculated because Astra's actual USD cost is not exposed. More economical future runs should avoid unnecessary repeated full-context delivery and review stable deltas once the architecture is settled.

Opus cost includes $3.056595 core model and $0.159167 exposed Haiku helper. It is tooling-reported list/API cost, not a subscription invoice. Core Opus input/output totals: 217,339 / 35,329 = 252,668 tokens; helper adds 159,019 tokens. Astra input/output totals: 1,968,263 / 17,803 = 1,986,066 tokens, including 1,875,328 cached input and reasoning within output. These reflect different harness/context delivery and are not comparable as an efficiency benchmark.

Elapsed active review totals: Astra 900.410 seconds; Opus 510.078 seconds. The reviews ran in parallel within each round; these totals are not wall-clock project duration. Astra supplemental start is corrected to the actual task_started event, 15:11:00.113Z, ending 15:14:16.297Z (196.184 seconds). The sealed reconciler report recorded the earlier metadata discrepancy; this telemetry correction resolves it without changing review findings.

Reconciler and focused correction-verification tokens/time are attached in metrics.json after completion; their USD remains unavailable. Prior A1 reviews remain separately measured in [the first cost report](../cost-value.md). The disclosed Opus workflow spend across A1 and this round is $4.914611; do not count repeated fixes across rounds as independent discoveries.
