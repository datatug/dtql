# Layered ACL MVP — design review packet

Status: A1 reconciled recommendations and H1–H7 approved by the user; A2 implementation plan and completed contracts independently reviewed, reconciled and correction-verified; awaiting final human dispatch sign-off. Design only; no MVP implementation is authorized by this packet. Author: Astra A, 2026-09-08.

DTQL owns the portable policy and authorization wire specifications. DALgo evolves its existing `access`, `dtql`, and `condeval` implementation. InGitDB, OpenVaultDB, and the DataTug daemon enforce their own policy sets. The browser displays and administers authorized policy surfaces; it is never an enforcement boundary.

The MVP is a real DataTug browser → DataTug daemon → OpenVaultDB HTTP → embedded InGitDB adapter vertical slice, with persisted policies at all three owners, a protected structured query and key-targeted UPDATE, multiple blockers, and Explain Access. It does not recreate InGitDB's removed HTTP server or turn the read-only query builder into a SQL mutation console.

## Reading order and deliverables

| Document | Purpose |
|---|---|
| [01-current-state.md](01-current-state.md) | Evidence, reusable code, discrepancies, gaps, baseline validation |
| [02-architecture.md](02-architecture.md) | Boundaries, composition, execution and race handling |
| [03-policy-format.md](03-policy-format.md) | Proposed DTQL policy document and normative semantics |
| [04-authorization-contract.md](04-authorization-contract.md) | Wire types, taxonomy, management and Explain contracts |
| [05-dalgo.md](05-dalgo.md) | Additive evaluator, enforcement and provider interfaces |
| [06-identity.md](06-identity.md) | Stable identities, bindings, actors, trusted propagation |
| [07-ingitdb.md](07-ingitdb.md) | Authoritative storage and adapter integration |
| [08-openvaultdb.md](08-openvaultdb.md) | Mounts, transport, composition and arbitrary sources |
| [09-datatug.md](09-datatug.md) | Daemon enforcement, policy UX and Explain Access |
| [10-security.md](10-security.md) | Threat model, diagnostic disclosure, administration |
| [11-acceptance.md](11-acceptance.md) | Fixtures, expected results, E2E and negative tests |
| [12-work-packages.md](12-work-packages.md) | Dependencies, contracts, tasks, tests and handoff gates |
| [13-decisions.md](13-decisions.md) | Alternatives, recommendations and human decisions |
| [14-future.md](14-future.md) | Bounded transaction compatibility and custom-provider seam |
| [15-contract-completion.md](15-contract-completion.md) | A2 exact evidence, sample, recovery and metadata contracts |
| [16-handoff.md](16-handoff.md) | Executable package cards, gates and validation ownership |
| [contracts/README.md](contracts/README.md) | Machine-readable specification schemas and golden vectors |
| [18-mask-supplement.md](18-mask-supplement.md) | User-requested collection/column masks and reviewed clarification |
| [17-approval-closure.md](17-approval-closure.md) | Approved A1 findings and A2 closure traceability |
| [reviews/rubric.md](reviews/rubric.md) | Identical independent review instructions and metrics |
| [audit-snapshots.json](audit-snapshots.json) | Exact local repository revisions audited |

Normative MUST/SHOULD language describes proposed implementation obligations, not current behavior. C1–C6 architectural choices are approved; the exact A2 contract revision remains subject to the implementation-plan review checkpoint. Examples are specification fixtures, not installed policies. This packet lives together in the standalone DTQL repository to give both reviewers identical cross-repository material. Owner-repository document updates and links are explicit work packages, not a claim that those repositories already adopted this design.

## Approval boundaries

Before parallel implementation, approve C1 policy semantics/version, C2 decision/request/admin wire shapes, C3 principal trust profile, C4 provider and enforcement capabilities, C5 mutation revision contract, and C6 supported query/write subset. See the decision log. Existing supported DALgo behavior remains under its existing version; new stricter behavior is negotiated through the DTQL ACL profile.

The independent reviews must use the same packet digest and rubric without access to each other's findings. A separate reconciler classifies findings and produces the human decision report and cost/value comparison. Design approval and permission to dispatch implementation are separate; this phase stops for human sign-off.

Current checkpoint: [reviewed implementation plan, both reviews and cost/value](reviews/round-2/README.md). No implementation has started.
