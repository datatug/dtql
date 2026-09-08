# Independent reconciliation — frozen packet A1

**Verdict: not ready for implementation; ready for bounded document correction and human approval.** Both reviewers support the existing architecture. Their different verdict labels do not represent an architectural disagreement: both require contract fixes before independent implementation.

This reconciliation recommends 17 accepted root findings, including five high-severity roots. “Accepted” means a reconciler recommendation, not human sign-off or implementation authorization.

Preserve the explicit user directions: OpenVaultDB serves remote InGitDB; DALgo and OpenVaultDB expose policy discovery; MVP includes plan, supplied IDs and bounded top-N dry runs; no-row results disclose admission conditions where authorized, with private/too-large references; policies default **public**, while private policy information is available only to the relevant owner’s policy admin.

## Evidence and verification

Reviewed both sealed reports, their run metadata, the rubric and frozen packet. Verified packet size **171,433 bytes** and SHA256:

`efd8dd40013888eee43ab8582e35b506d9fbce9e61a8e08d21a4c6b004821b94`

The separately read packet files match the frozen input, ignoring trailing whitespace. Disputed DALgo claims were checked through read-only Git object reads at audited SHA **`6b0be09b95bd3afc90309763671791e17264dffe`**, covering `access/principal.go`, `policy.go`, `condition.go`, `session.go` and `fields.go`. No files were edited, implementation performed or tests executed.

Two source corrections materially affect the review:

- **O-05 is incorrect.** `PrincipalPolicySet.effective` already calls `prefixRuleNames` before compiling the union; the existing comparator therefore already sees qualified names. The proposed qualification does not introduce the claimed migration change. Preserve the existing comparator and differential tests; do not introduce unqualified or declaration-order precedence.
- **O-04’s suspected query-field failure is unsupported by the source.** `policy.go` retains `Decision.Writes` when an unconditional terminal removes row residuals. `condition.go:queryFields` collects field bounds from every alternative and terminal, and `session.go:authorizeQuery` uses them. `fields.go` implements intersection. Point reads instead use `decidingFields`, selecting the first matching alternative or terminal. Clarifying this distinction and adding vectors is useful; changing it to a union is not justified.

## Root finding dispositions

Closely related findings remain separate when they require different corrections. For example, duplicate-key candidate semantics and the HTTP batch envelope are distinct. Likewise, missing sampled-result identities and the meaning of `enforced` require different fixes.

| Root / review IDs | Classification; severity | Disposition, rationale and work-package action |
|---|---|---|
| **R01 — A-01** | Complementary; **high** | **Accept.** Specify how `(realm,kind,id)` reaches bindings and identity-valued authorization parameters. Raw `fmt.Sprint(ID)` matching cannot preserve kind separation. Define human/nonhuman direct bindings, realm mapping and `currentUser` behavior without replacing established hosted user IDs. Add same-ID/different-kind and cross-realm vectors. **WP1/2/5; C1/C3 blocker.** |
| **R02 — A-02** | Complementary; **high** | **Accept.** Separate operation IDs do not prevent repeated normalized targets. Independent pre-image checks can admit a forbidden combined final image. Recommend rejecting duplicate normalized record targets in execution batches before data access. Define whether dry-run operations are independent assessments; never imply sequential simulation without specifying it. Add the `x/y` counterexample. **WP1/3/4/5; C2/C5/C6 blocker.** |
| **R03 — A-03** | Complementary; **high** | **Accept.** Complete the authoritative C2 schema: sampled key association, template/result identity, reduction, zero sample, plan without mutation, restriction representation and `allOf`. Include complete mixed-sample, private and oversized-response goldens. This preserves requested modes rather than postponing them. **WP1, consumed by WP2/7; C2 blocker.** |
| **R04 — A-04** | Complementary; **medium** | **Accept.** Define normal-query default/max rows, offset bounds, buffer bytes and bound-exceeded behavior. Dry-run limits do not bound ordinary queries. An oversized record defeats a row-count-only buffering guarantee. **WP1/3/5/6; C6 blocker.** |
| **R05 — A-05** | Complementary; **medium** | **Accept.** Existing in-memory rollback does not establish crash recovery across policy files, manifest, Git state and snapshot publication. Specify a concrete commit/recovery sequence and process-failure tests. Recovery from committed state is already compatible with the proposed design; choose and document its actual steps. **WP4/5/6 stores; implementation-design gate, not necessarily portable contract freeze.** |
| **R06 — A-06, O-01; O-09 subsidiary** | **Consensus**, with complementary local/remote evidence; **high** | **Accept with narrowed remedies.** Specify trusted in-process evidence access and transaction lifetime separately from remote authorized evidence. Remote consumers must distinguish withheld fields from authoritative absence, with coverage and revision tied to one snapshot. Resolve dependencies across potentially deciding alternatives. O-09 repeats the hidden-evidence limitation and adds an authoring warning opportunity; do not count it as another security discovery or require a universal static cross-layer permission solver. **WP1/3/4/5/6; C4/C5 blocker.** |
| **R07 — O-02** | Complementary; **high** | **Accept.** Policy-reference diagnostics and protected-row existence/predicate facts need an explicit disclosure boundary. The packet’s general non-disclosure rule conflicts with T07 granting only reference diagnostics while exposing a protected-row predicate failure. Define equivalent missing/denied public projections and the precise grant for enhanced row facts. Preserve the three-blocker mission by provisioning an appropriate narrowly scoped diagnostic identity. **WP1/3/5/8; C2 blocker.** |
| **R08 — O-03** | Complementary concern; **conflicting remedy**; **medium** after verification | **Accept only contract clarification.** Document cross-hop reference treatment, trusted adapter assumptions and how upstream obligations are verified before publication. The packet already requires fail-closed behavior when lower layers forbid required fields, and rejection of inconsistent adapter results. Reject a default exemption for caller-supplied `constraints[]`: lower-owner hidden-field restrictions remain authoritative. An echoed attestation alone does not prove correct filtering. Add hidden-field and dropped-predicate tests; preserve before-paging enforcement. **WP1/3/5/6; C6 clarification gate.** |
| **R09 — O-04** | Incorrect/unsupported suspected defect; **low** clarity value | **Reject as a contract-freeze defect.** Source preserves conservative query bounds. Optional C1 clarification and get/query/update vectors should state first-decider point reads versus conservative query intersection. Reject the proposed union change. **WP1/2 optional clarification.** |
| **R10 — O-05** | **Incorrect**; no accepted severity | **Reject.** Qualification already occurs in legacy DALgo. No new precedence decision is needed on this basis. Keep existing migration tests and rename warning. **WP1/2: preserve, not redesign.** |
| **R11 — O-06** | Optional boundary clarification; **low** for local MVP | **Defer additional architecture.** Packet already excludes hostile repository/filesystem writers. Make the practical consequence explicit if editing the threat model: direct policy-file write authority bypasses runtime administration. Protected Git workflows/signing/new stores belong to future cloud/Git collaboration design. Do not claim every constrained PR contributor automatically has unrestricted policy-write authority. **WP9/14 follow-up.** |
| **R12 — O-07** | Complementary; **medium** | **Accept.** Define classification authority, create/import default, PUT omission, mismatch and canonical export/import behavior. Recommend create/import default public; replacement omission preserves current classification; explicit change requires owner policy-admin authority. Canonical output should preserve the effective classification rather than silently strip it. **WP1/2/4/5/6; C1/C2 blocker.** |
| **R13 — O-08** | Complementary; **medium** | **Accept.** A public policy can still be unavailable to a requester under ordinary owner authorization. Define the appropriate omission reason and independently gate expression, field names and policy references. A shared disclosure matrix must preserve public-default and private-admin-only rules. Clarify the ambiguous “Only … admin” sentence in 10 so it clearly refers to private policies. **WP1/5/7; C2 blocker.** |
| **R14 — O-10** | Complementary to R02; **medium** | **Accept.** Specify per-item revisions, operation identity, transaction boundary, whole-batch rejection and HTTP conflict/denial envelope. Maintain bounded supported batches and T16 rather than silently removing them. Unsupported adapters must advertise rejection. **WP1/3/4/5; C2/C5 blocker.** |
| **R15 — O-11** | Complementary; **medium** | **Accept focused conformance strengthening.** Existing generic tests are planned, but exact cross-engine semantics need vectors for numeric/string comparison, null/missing, type errors, `In`, ordering, nested fields, captures and identity mapping. Execute applicable vectors at the adapter boundary. Do not require every vector to enlarge the main browser fixture or add another remote deployment. **WP1/2/3/4/5/8; fixture gate.** |
| **R16 — O-12** | Complementary; **medium** | **Accept scope clarification.** Define which schema metadata can be disclosed and which schema/DDL routes reject on enabled mounts. Existing “review escape routes” language needs an explicit operational result. Keep policy metadata inaccessible through generic data/schema APIs. No new DDL policy language is necessary for MVP. **WP1/4/5/7; C6 boundary gate.** |
| **R17 — O-13** | Optional sequencing; **conflicts with requested scope** | **Reject proposed MVP reduction.** Shipping only plan/IDs/opaque references would omit user-requested top-N and disclosable no-row conditions. A preliminary single-owner milestone can help internal sequencing, but cannot replace the three-owner acceptance deliverable. Simulation deferral would require an explicit scope decision; it is not automatically accepted here. **WP12 planning only.** |
| **R18 — O-14** | Optional/future; **low** | **Defer grammar expansion/reservation.** Positive-only admissibility is already explicit. Documentation may add examples of unsupported negative expressions and safe positive alternatives. Do not reserve operators merely to avoid a hypothetical future version change. **WP9/14.** |
| **R19 — O-15** | Duplicate of existing documented decision; **low** | **Defer as a standalone finding.** D15 already states that admitted operations may complete and later admissions see edits. An operator-facing restatement is useful; request-deadline details belong with accepted execution bounds. No new revocation architecture or approval choice is established. **WP9, coordinated with R04/R22.** |
| **R20 — O-16** | Complementary to schema completeness; **low** | **Accept.** Define `restrictions[].enforced` for plan, inspect and successful execution. False must not mean irrelevant; dry run cannot imply that a later mutation is authorized. Renaming or replacing the boolean is optional if precise semantics suffice. **WP1/2/7; small C2 clarification.** |
| **R21 — O-17** | Complementary; **low** | **Accept.** Enumerate the referenced 13 mission proofs or replace the unexplained numbering with self-contained acceptance requirements. Preserve traceability to the user’s intended proofs. **WP1/8 documentation gate.** |
| **R22 — O-18** | Complementary; **low** | **Accept bounded operational correction.** Specify lock-wait cancellation/deadline handling and its result mapping, then add one meaningful contention case. A new throughput SLO or lock architecture is unnecessary. **WP3/4/5/8; before integration acceptance.** |
| **R23 — O-19** | Optional; **low/future** | **Defer negotiation expansion.** Exact-version rejection and no unrestricted fallback already establish a safe MVP boundary. Advertising supported versions is useful compatibility metadata, but absent negotiation is not an implementation blocker. **WP1/9 optional.** |
| **R24 — O-20** | Complementary to R01; **low** | **Accept bounded identity/diagnostic detail.** C3 already calls for an explicit bootstrap principal; specify its stable owner-local identity construction and lifecycle. Distinguish actor capability failure from subject policy failure in safe attribution. This is different from R01’s binding collision defect. **WP1/5; resolve before auth implementation.** |

## Human decisions and trade-offs

The following decisions need concrete document recommendations at the checkpoint. Existing user directions must not be reopened as if undecided.

1. **Typed binding semantics:** approve the exact principal-kind mapping and identity parameter representation. Preserve direct hosted UID identity; avoid untyped matching.
2. **Protected-row diagnostics:** decide which owner-scoped authority permits existence/predicate facts and provision T07 accordingly. Reference-only diagnostics should not silently gain protected-row inspection. Policy administration must not automatically grant data visibility.
3. **Supported batches:** approve duplicate-target rejection, per-item CAS and all-or-none behavior for the reference adapter. Sequential virtual updates are a larger alternative, not a prerequisite.
4. **Execution limits:** approve advertised/server-enforced row, offset, byte and deadline limits. Numbers need to be concrete before dependent implementations diverge.
5. **Recovery guarantee:** retain automatic recovery to an old or new complete generation and specify the mechanism, or explicitly weaken T14 to fail-closed startup requiring repair. These are different availability guarantees.
6. **Schema boundary:** approve the allowed metadata projection and explicitly unsupported DDL behavior for ACL-enabled mounts.
7. **Original C1–C6 choices:** approve namespace, field allow-list behavior, canonicalization, self-hosted fixture profile, per-owner snapshots and readable remote evidence/CAS limitations.

Recommended conflict resolutions:

- Keep existing DALgo precedence and conservative query fields. O-04/O-05 provide no basis for a semantics redesign.
- Keep remote hidden-field fail-closed behavior. Trusted in-process evidence access is a separate mechanism; it does not justify a cross-owner raw-data or caller-reference exemption.
- Keep `public`/`private` and the requested MVP modes. Clarify their UI descriptions and owner authorization; do not rename or remove them merely to reduce perceived complexity.
- Keep the real three-owner browser vertical as the delivery gate. Intermediate milestones are implementation sequencing, not scope substitution.

## Accepted contract-fix checklist

Before downstream contract dispatch:

- **WP1:** resolve typed binding/bootstrap contracts; complete sample and restriction schemas with full goldens; define diagnostic disclosure and visibility replacement semantics; specify batch identities/CAS/atomicity and duplicate rejection; specify evidence completeness; define query bounds, remote verification obligations, schema boundary and `enforced`; make mission proofs self-contained.
- **WP2:** preserve verified legacy precedence and point/query field semantics; implement only the approved typed adapter and shared DTO/evaluator changes; consume differential and semantic vectors.
- **WP3:** specify trusted evidence lifetime, dependency completeness, duplicate-target validation, pre-paging restrictions, bounded verification and cancellation behavior.
- **WP4/5:** specify policy publication recovery, embedded evidence access, reference-adapter batch transaction semantics, remote coverage/revisions, safe missing/denied mappings and schema routes.
- **WP6:** preserve principal context; enforce remote evidence and query bounds; retain lower inspection after upper static denial.
- **WP7:** render complete sample identities, distinct omission reasons, precise enforced/conditional states and safe actor/subject attribution; preserve requested modes.
- **WP8:** add focused regression cases for accepted counterexamples and adapter semantics, plus crash and contention cases. Run the existing real vertical acceptance gate.
- **WP9:** carry verified source corrections and operator limitations into owner documentation.

The document author may now prepare these proposed corrections with a revision ledger. This report does not authorize MVP code.

## Reviewer value and overlap

**Counting method:** 26 submitted IDs become **24 semantic roots**. O-01 and O-09 share the evidence-availability root; A-06 independently raises that same root. All other IDs remain separate because their required corrections differ. General agreement about the architecture is not counted as a finding.

| Metric | Astra | Opus |
|---|---:|---:|
| Submitted finding IDs | 6 | 20 |
| Roots after internal deduplication | 6 | 19 |
| Accepted roots | 6 | 12 |
| Unique accepted roots | 5 | 11 |
| Accepted critical roots | 0 | 0 |
| Accepted high roots | 4 | 2 |
| Unique accepted high roots | 3 | 1 |

There is **one independently shared root**, R06.

- Root Jaccard overlap: **1 / (6 + 19 − 1) = 1/24 = 4.2%**.
- Smaller-review coverage overlap: **1 / min(6,19) = 1/6 = 16.7%**.
- Accepted-root Jaccard: **1 / (6 + 12 − 1) = 1/17 = 5.9%**.
- Combined accepted roots: **17**, including **five high**, no critical.
- These deliberately conservative overlap counts do not equate “both discussed identity,” “both discussed batches,” or “both discussed C2” with discovery of the same defect. Broader thematic overlap is substantially greater.

**Astra’s distinctive value:** typed binding collisions, duplicate-key candidate semantics, sampled-result/schema incompleteness, ordinary-query bounds and crash publication recovery. Its shared evidence finding contributes a particularly concrete redacted-versus-absent fallback counterexample. No assessable false-positive root was found among its six findings; narrower coverage means it missed several Opus disclosure and transport corrections.

**Opus’s distinctive value:** protected-row diagnostic leakage, visibility PUT semantics, authorization-specific omission reasons, batch transport/CAS shape, adapter semantic vectors, schema boundaries and smaller operational/attribution clarifications. Its breadth was useful, but triage was essential. O-05 is a verified false positive; O-04’s suspected implementation contradiction is unsupported. O-03 recommends an exemption that conflicts with the deliberate remote boundary; O-13’s reduced delivery would omit explicit user requirements. O-15 mostly repeats an existing decision. Opus candidly marked several source inferences for verification, which made reconciliation easier.

**Would Opus alone likely suffice?** It would likely identify that contract freeze was premature and provide a useful broad review. It would not be a sufficient substitute for the combined result here: Astra supplied three unique accepted high-severity roots, including two concrete authorization/correctness counterexamples and a wire-contract blocker. A broad review plus author verification might eventually discover them, but this run provides no evidence that it would.

**Incremental Astra value:** five unique accepted roots, three high and two medium, plus independent corroboration and a stronger remote evidence counterexample. Its monetary efficiency cannot be measured because cost is unavailable.

**Future workflow recommendation, based on this project only:** retain a broad architecture/disclosure reviewer and an independent adversarial contract reviewer, followed by source-backed reconciliation. Supply the full principal-union and field-enforcement call chain in future source excerpts. This is evidence for complementary roles on this packet, not a universal model ranking.

## Measured usage and cost

Telemetry comes from author-side run artifacts, superseding reviewers’ truthful statements that metrics were unavailable inside their review turns. Costs below are **reported list-price usage costs, not invoice or subscription costs**.

| Item | Astra independent review | Opus primary model |
|---|---|---|
| Exact model | `gpt-6-astra` | `claude-opus-5` |
| Start | 2026-09-08 13:39:06.565 UTC | CLI run: 2026-09-08 13:39:25.493555 UTC |
| End | 2026-09-08 13:46:19.629 UTC | CLI run: 2026-09-08 13:49:11.122227 UTC |
| Elapsed | 433.064 s | CLI run: 585.629 s |
| Input, including cache categories once | 461,622 | 63,258 |
| Cached read input | 403,584, included above | 0 |
| Cache creation input | 0 reported | 63,256, included above; one-hour cache |
| Other input | 58,038 | 2 |
| Output | 8,132 | 40,869 |
| Reasoning/thinking, included in output | 4,879 | 28,416 |
| Total input + output | 469,754 | 104,127 |
| Reported cost | **Unavailable** | **$1.654295** |

Astra input is cumulative across review turns and includes repeated cached context. Its output already includes reasoning. Opus’s cache creation is an input category, not a cache hit and not additional output; its thinking is already included in output. The raw input numbers should therefore not be compared as unique packet sizes.

The Opus CLI run also reports ancillary **`claude-haiku-4-5-20251001`** usage:

- Input **44,474**, output **16**, no cache reads/creation reported.
- Reported cost **$0.044554**.
- Complete Opus CLI run: input **107,732**, output **40,885**, total **148,617** tokens.
- Complete reported cost: **$1.698849** = $1.654295 Opus + $0.044554 Haiku.

Using the complete Opus run cost:

- Cost per accepted root: **$1.698849 / 12 = $0.1416**.
- Cost per unique accepted root: **$1.698849 / 11 = $0.1544**.
- Astra cost per accepted/unique root: **unavailable**, not zero.
- A dollar comparison of incremental Astra value is therefore unavailable.

These are descriptive counts, not severity-weighted productivity measures. They depend on the disclosed root/disposition method and one project. Available run metadata records successful completion; no separate retry-cost amount was supplied. The complete CLI cost includes the reported ancillary model, but should not be represented as a subscription invoice.

**Reconciler telemetry:** the author will attach exact model, timing, cumulative usage and available cost after this turn completes. None is estimated here.
