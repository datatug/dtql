# Final document corrections after independent A2 + S1 reconciliation

Frozen A2 and S1 inputs/reviews remain unchanged. Current numbered documents/schema fixtures incorporate the accepted recommendations; no MVP code was written. This ledger records author actions for separate focused verification, not an assertion that the original frozen packet passed unchanged.

| Reconciled change | Applied final contract / work package |
|---|---|
| A2-01 / O2-18 session portion | 15 splits InspectionSession/ExecutionSession and admission APIs; pure plan bypasses coordinator; trusted plan immutable; protected dynamic worker rejected before invocation; internal raw transaction only, legacy profile unchanged; 11 regression + 16 WP3b/4b |
| A2-02 / O2-06 | 15 selection/template diagnostic domain, collision-skipping sN allocation, cross-hop remapping, sample selection denial reducer; sample.templateOperationId/selectionResource schema; complete selection denial/unavailable/empty/collision goldens |
| A2-03 | 15 internal generation vs per-document ETags; non-admin whole-set layer revisions always omitted; public CAS merges current private changes; 11/semantic vector |
| A2-04 | 12/16 start vs completion dependencies; real WP2a/b required to finish store/transport, G3 circular wording removed |
| O2-03 narrow | 03 whole candidate rejects unsupported collectionGroup/opaqueQuery selectors; representable unrestricted truncate rules retained with execution unsupported |
| O2-04 | 15 non-admin omission not_authorized, never private |
| O2-05 clarification | 15 plan residuals restrictions, not missing evidence; no row_evidence_required in plan |
| O2-07 | Evidence resource.columns forbidden in request/response; requiredFields sole exact coverage; schema/goldens corrected |
| O2-08 narrow | 15 remote absence-dependent set unsupported; candidate-only insert preserved |
| O2-09(a) | 15 existing valid digest reused, inconsistent collision rejected; process-kill acceptance retained. GC remains follow-up; no new cleanup/loader subsystem required. |
| O2-11 | 08 host batch mapping, per-item IDs/CAS/conflict; batch schema definitions |
| O2-12 | 04 strict query/sample decoder depth16/nodes4096/body1MiB, no aliases/tags/multiple/duplicate |
| O2-13 narrow | 08 WP5b registered capability matrix/defaultfalse, SQLite static/read smoke; proven row pushdown not incorrectly forbidden |
| O2-14 | 03 numeric Unix milliseconds now; list params In only; vectors |
| O2-17 | 04 evidence HTTP route; 11/12/16 mandatory T01–T21/V01–V08 |
| AS-01 | 18 projectable parent vs complete evidence vs scalar query reference; possible excluded subtree rejects parent evidence even child absent; descendant alternative; fixtures |
| Astra whitespace | 18 new masks reject surrounding whitespace, legacy trim unchanged; semantic vector |
| OS-02 / OS-06 schema | Exact collectionMask/fieldMask, mutual exclusion, restriction mask/reference, collection reason code, tightened nameMask; two canonical YAML fixtures and negative masks |
| OS-03 | 18 policy-read required for mask literals, admin additionally for private; otherwise opaque, never weakened include-only constraint |
| OS-04 | 18 resolved failed gate vs unsupported collection code and root-only migration wording |
| OS-08 | 18 explicit direct InGitDB/OVDB/DataTug T21 matrix + multiple owners/alternatives |

Optional low-cost precision adopted: mask arrays explicitly sorted/deduplicated with canonical fixtures; simple JSON Schema allowed converse; standard deny-column remains explicit fields list and advanced masks carry future-column warning; legacy address/address.* differential; affected-path/array/projection regression detail. Rejected security claims do not trigger replacement architectures. No new standalone grant/auth service, field encoding, route-only policy kind, native effects engine, logical transaction system, generation GC or custom ACL service was added.

Validation: 19 structural JSON fixtures meet expected valid/invalid classification; two canonical YAML fixtures parse to their JSON documents; current local document links resolve. Semantic runtime vectors and new E2E tests are specifications, not implementation tests claimed passed. A1/A2/S1 frozen input hashes are preserved independently of current-file changes.
