# Approval and review closure ledger

2026-09-08: user approved all suggestions at the A1 reconciled checkpoint and requested the implementation plan with independent Astra/Opus review and separate reconciliation. This is interpreted as the reconciler's **accepted** recommendations plus author resolutions H1–H7, not raw findings rejected as incorrect or out of scope. It does not authorize MVP implementation. Later same-turn direction adds execution-class separation and procedure masks (User_* inclusion; all except sys_* exclusion) as C7.

| A1 root / approval | A2 concrete disposition | Verification |
|---|---|---|
| R01 / H1 | Typed realm/kind/ID, human-only users/currentUser, explicit nonhuman roles/groups | 03/06/15; V01 |
| R02 / H3 | Unique normalized batch targets, per-item IDs/CAS, one transaction, independent dry runs | 15; V04 |
| R03 | Closed schema, sample row/template identity/reducer, restriction variants/ref validation | contracts + 15; V07 |
| R04 / H4 | Exact query/byte/offset/deadline limits and no partial execution response | 15; V08 |
| R05 / H5 | Immutable generations, atomic activation, committed Git pointer and process-crash recovery | 07/15; V05 |
| R06 (A06/O01/O09) | Registered private evidence session distinct from authorized remote field coverage; absence != withheld | 15; V03 |
| R07 / H2 | Concrete protected-key diagnostic grant separate from policy reference/admin permission | 10/15 + T07/V06 |
| R08 | Preserve HTTP field authority, before-page constraints, response verification; no arbitrary hidden-field exemptions | 15 + T10 |
| R12 | PUT omitted visibility unchanged; canonical emits effective visibility | 03 + T14 |
| R13 | not_authorized omission and separate facts/references/data disclosure | 04/15 + V06/V07 |
| R14 | Batch atomicity and per-item CAS details | 15 + V04 |
| R15 | Scalar/type/null/path/identity golden specification matrix | contracts + V01/V02 |
| R16 / H6 | Separate filtered schema read, reject DDL on protected data routes | 15 + V08 |
| R20 | enforced means applied at actual boundary; false for every dry run | 15 + V07 |
| R21 | All 13 mission proofs explicitly enumerated | 11 |
| R22 | Lock waits consume same deadline | 15 + V08 |
| R24 | Stable bootstrap ID; requester/actor/subject attribution | 06 + V01 |
| H7 | Existing hierarchy/fields, namespace, semantic canonicalization, selfhost fixture, snapshots/CAS retained | 03/06/15 |

R09/R10 unsupported code-change suggestions remain rejected: current DALgo already qualifies rule-set IDs and conservatively intersects query fields. R11 Git-owner trust, R18 negative leaf operation grammar, R19 standalone revocation redesign, R23 optional version negotiation enhancements remain deferred. R17 removal of user-requested sampling remains rejected. Re-review may identify new concrete defects; it should not count these resolved/deferred preferences as new findings without new evidence.

C7 addition is newly requested scope, included in both reviewers' identical packet: control/representation and pure mask evaluation are MVP; native SQL/GraphQL/procedure effects execution stays unsupported. No new native parser or stored-procedure execution subsystem is planned. Architecture approval does not mean a matching inclusion mask bypasses table/row/column or lower-owner restrictions.
