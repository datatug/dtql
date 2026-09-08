# Human checkpoint — design and implementation plan reviewed

Historical A1 checkpoint. **The user subsequently approved these H1–H7 resolutions and reconciler-accepted suggestions and requested an A2 implementation-plan review.** Implementation dispatch remains unauthorized. No MVP implementation has started. The frozen A1 packet includes both architecture/specification and [the implementation work packages](../12-work-packages.md); both independent reviewers and the separate reconciler reviewed that same plan with the design.

## Review material

| Artifact | Result |
|---|---|
| [Design packet](../README.md) and [implementation plan](../12-work-packages.md) | 14 documents, current-state audit, proposed C1–C6 contracts, WP0–WP9 and E2E acceptance |
| [Astra B review](astra-review.md) | Not ready for implementation; six bounded contract/design findings; supports the architecture |
| [Opus review](opus-review.md) | Ready with changes; 20 submitted findings, including three high findings; requires contract fixes before dispatch |
| [Independent reconciliation](reconciliation.md) | 17 accepted root recommendations, five high; verifies disputed code claims; no replacement architecture recommended |
| [Measured cost/value report](cost-value.md) | Actual exposed telemetry, semantic overlap and reviewer contribution; missing costs explicitly unavailable |
| [Frozen input](input-A1.txt), [manifest](manifest-A1.json), [rubric](rubric.md) | Identical material, independent parallel reviews; packet SHA256 `efd8dd40013888eee43ab8582e35b506d9fbce9e61a8e08d21a4c6b004821b94` |

The reviewers' different headline verdicts are not a disagreement about the architecture: both require corrections before independent implementation. Original reviews and the complete frozen A1 input remain unchanged; original standalone documents are also recoverable from commit `dcd3bc0`. Current design files evolve as A2. The approval applies to accepted reconciler recommendations and H1–H7, not rejected or deferred raw findings.

## What stays settled

Retain DTQL contract ownership and reuse DALgo's existing ACL engine. Every owner enforces its own policies. OpenVaultDB is the remote InGitDB server. Retain user-requested policy discovery, plan/row-ID/top-N dry runs, bounded disclosable conditions, **public policy default** and private-policy admin-only visibility. Do not replace the three-owner vertical acceptance with a single-owner deliverable.

Reconciliation rejected the claimed new rule-qualification migration conflict: DALgo already qualifies rule IDs before sorting. It also verified the existing conservative query-field intersection. Do not change those semantics in response to the unsupported findings. Do not exempt arbitrary remote predicates from lower-layer field restrictions merely to simplify push-down.

## Concrete recommendations for human decision

The following are **Astra A's proposed resolutions**, informed by the separate reconciler. They are decision proposals, not applied C1–C6 amendments or a claim that the reviewers approved these exact new details.

| Decision | Recommended resolution | Alternative / consequence |
|---|---|---|
| H1 Typed identity | Keep owner realm + principal kind + existing stable ID in verified context. `bindings.users` matches only human users in the configured realm; nonhuman identities use explicitly assigned role/group bindings in this MVP. `currentUser` resolves only for humans, otherwise fails closed. Test same ID across kinds/realms. | Adding typed direct nonhuman subject bindings now is possible but requires an extra C1 shape. Never feed raw nonhuman IDs into user bindings. |
| H2 Protected-row diagnostics | Separate policy-reference permission from protected-row facts. Require an explicit owner-scoped, key-bounded `access:inspect-protected` grant to disclose existence/predicate failure on an unreadable row; add it to T07's diagnostic fixture. Ordinary missing/denied rows use the same safe projection. Private policy identity/text remains admin-only even with this grant. | Coalescing every protected-row fact is safer but makes T07's detailed row blocker unavailable except to an appropriately authorized diagnostic/admin identity. No blanket data-read grant is proposed. |
| H3 Batch semantics | Keep bounded batches on capable reference adapters: unique normalized record targets only, per-item operation IDs and revisions, one storage transaction, all-or-none preflight/CAS/mutation. Dry-run operations are independent assessments against a snapshot, not a sequential change-set simulation. | Supporting repeated writes to one key needs explicit virtual-image semantics; defer that work. |
| H4 Execution limits | Propose default/max 1,000 returned query rows, max offset 10,000, max 8 MiB buffered result, 10-second execution deadline; retain 2-second dry-run budget and sample maximum 100. Bounds are advertised and enforced; overflow fails before partial success. Lock waits consume the request deadline. | Different limits can be approved; they must be concrete and compatible across adapters. A client timeout after commit can leave an uncertain outcome and must not imply automatic safe retry. |
| H5 Policy-store recovery | Retain automatic recovery to a complete committed generation. Specify immutable generations/atomic activation for filesystem stores, and explicit Git commit/activation/recovery ordering for InGitDB before storage implementation. Inject process crashes, not just returned errors. | Weaker fail-closed startup requiring operator repair must be stated by narrowing T14; do not call in-memory rollback crash-atomic. |
| H6 Schema boundary | Use separately authorized, field-filtered schema discovery for ordinary principals. Explicitly reject DDL on the ACL-enabled MVP data surface; fixture schema provisioning occurs before enabling ACL. Full metadata administration is separate from ordinary policy/read grants. | Treating field names as nonconfidential is another possible product choice, but it weakens the stated confidentiality boundary. No new general DDL policy language is recommended. |
| H7 Original contract choices | Approve proposed DTQL namespace, existing hierarchical/field semantics, semantic round-trip without comment preservation, self-hosted authenticated fixture, per-owner admission snapshots and readable remote evidence + whole-record CAS limitation. | Changes here must update design, schemas/vectors and work packages together before parallel implementation. |

Additional mechanical document corrections do not need a new architecture: complete sampled per-row identities and restriction variants; define `enforced`; define visibility omission on replacement as unchanged and canonical serialization as explicit effective classification; add an authorization-specific omission reason; make all 13 mission proofs self-contained; and add the accepted predicate/evidence/identity/batch regression vectors.

The evidence seam needs an explicit contract correction alongside H1–H7: owner-authorized in-process enforcement evidence is distinct from remote requester-readable evidence. Bind complete pre/post images and revisions to one short operation/transaction; distinguish authoritative field absence from redaction. Retain fail-closed behavior when remote evidence is unavailable. The exact interfaces and complete wire shapes must be written and checked before C1–C6 freeze, not improvised by parallel agents.

## Next authorized phase after decisions

Apply approved **document** corrections to design and implementation plan together, record A2 amendments, complete/check contract schemas and golden examples, and obtain a focused independent verification of the accepted high-risk fixes. Then freeze the approved C1–C6 revision and prepare Corpus handoff. This checkpoint requests decisions for that finalization; it does **not** dispatch implementation automatically.

No implementation agents have been dispatched. Baseline DALgo tests passed; proposed MVP/E2E tests remain specified work. The design worktree is retained for review and finalization.
