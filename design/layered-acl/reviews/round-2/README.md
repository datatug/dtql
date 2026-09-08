# Human checkpoint — reviewed implementation plan

**Historical A2/S1 checkpoint. The subsequent [A3 scoped-mask amendment](../../19-scoped-mask-stages.md) changes mask semantics and requires an updated contract review before implementation.** Implementation has not started. The user approved A1 suggestions, then requested this implementation-plan phase with the same two-model review. Both models independently reviewed the same A2 packet and the same subsequent collection/column-mask supplement. A separate agent reconciled findings and verified the applied document corrections.

| Deliverable | Reviewable artifact |
|---|---|
| Architecture and contracts | [Packet index](../../README.md), [exact completion](../../15-contract-completion.md), [schemas/goldens](../../contracts/README.md) |
| Executable plan | [Work packages](../../12-work-packages.md), [package cards/dependencies/gates](../../16-handoff.md) |
| User-requested masks | [Procedure/class controls](../../03-policy-format.md), [collection/column masks](../../18-mask-supplement.md) |
| Actual E2E implementation acceptance | [T01–T21, V01–V08, all 13 mission proofs](../../11-acceptance.md) |
| Independent Astra review | [Main](astra-review.md), [mask supplement](astra-supplement-review.md) |
| Independent Opus review | [Main](opus-review.md), [mask supplement](opus-supplement-review.md) |
| Separate reconciliation | [Finding classification and model evaluation](reconciler-review.md) |
| Applied corrections | [Closure ledger](correction-ledger.md), [independent correction verification](reconciler-verification-review.md) |
| Usage/value | [Cost/value report](cost-value.md), [actual exposed metrics](metrics.json) |

Astra's main verdict was not ready: it identified transaction-admission, sample-selection and revision-privacy gaps, plus package completion dependencies. Its supplement added the distinction between projected parent data and complete evidence. Opus's verdict was ready with changes: it added useful transport/parser/fixture checks, while the reconciler rejected or narrowed unsupported security claims. The reconciler recommended 21 root changes/clarifications, rejected seven and deferred seven. Those accepted changes are now applied; the separate correction verifier found **no remaining concrete contract blocker**.

The final plan supports procedure, collection and column inclusion/exclusion masks with exclusions winning within a mask set and restrictive composition across owners. Existing DALgo hierarchy remains. Public policies remain the default; private policies remain owner-admin-only. Protected native SQL/GraphQL/procedure effects execution is explicitly unsupported in this MVP even if a mask matches; DTQL route gating and mask evaluation are proven by planned tests. Standard deny-column editing retains explicit allow-lists; advanced wildcard masks explain their future-column behavior.

No unresolved architecture decision remains from this review. The documented protected-profile limit is that dynamic transaction callbacks reject before worker invocation; bounded predeclared operations/batches use the shared coordinator. Legacy behavior remains under its existing profile. Runtime implementation still must prove the prescribed adapter, privacy, crash, revision and browser tests.

Validation actually performed: 19 structural fixtures, both canonical YAML-to-JSON examples, source/packet hashes and local links. The separate verifier independently checked all 46 final manifest entries and the fixtures. No claim is made that the new runtime ACL/E2E tests already pass.

Opus main+supplement reported $3.215762 including its exposed helper; Astra and reconciler USD are unavailable. Astra supplied fewer but deeper contract findings; Opus supplied broader bounded checks. For this security-sensitive plan, both added useful evidence, but financial ROI cannot be calculated without the missing costs. Full counts/limits are in the cost report.

## Integrity and authorization boundary

Frozen [A2 input](input-A2.txt), [S1 input](input-A2-S1.txt) and their manifests preserve exactly what both reviewers saw. [Final corrected input](input-final.txt) and [verified manifest](manifest-final.json) preserve what the correction verifier checked. The only subsequent changes to those verified normative files are the verifier-requested editorial mask-variant cross-reference in 15 and readiness/navigation text in the index; [candidate manifest](manifest-candidate.json) records the current candidate files. Original A1 input/reviews remain in the parent review directory and commit dcd3bc0.

This is the human checkpoint required by the original review workflow. Approving this plan can freeze the candidate contracts and authorize the later Corpus handoff. No implementation agent is dispatched automatically, and no MVP implementation or owner-repository code has been changed in this phase.
