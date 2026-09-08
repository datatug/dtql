# Current-state audit

Audit date: 2026-09-08. Evidence is from the clean local revisions in [audit-snapshots.json](audit-snapshots.json), not a claim that every local clone is the latest remote release. DTQL's isolated branch was created from fetched `origin/main`. All paths below are relative to the named repository; inspect them at its recorded SHA. Tests were inspected as well as implementation. No MVP code was changed.

## Repository and evidence map

| Repository | Evidence inspected | Reuse and limits |
|---|---|---|
| `datatug/dtql`, `datatug/dtql-go` | Root READMEs | Specification/toolkit aspirations; essentially placeholders. No competing mature policy parser found here. |
| `dal-go/dalgo` | `docs/access-policies.md`; `spec/features/access-policies/{README.md,row-level-conditions/README.md,principal-bindings/README.md,field-patterns/README.md}`; `access/*.go` | Existing hierarchical capability model, YAML/JSON codecs, default deny, principal rule-set bindings, row predicates, post-image checks, field allow-lists, secured DB/session wrappers. Evolve this. |
| `dal-go/dalgo` DTQL | `dtql/{shapes.go,serialize.go,deserialize.go,query.go,schema.go}`, `dtql/examples/comparison.dtql.yaml`, `spec/plans/2026-06-05-dtql.md` | Working query interchange uses YAML `from`, `columns`, `where`, `orderBy`, `limit`, `offset`; typed expression nodes `field/value/values/param`, comparisons and `and/or`. A bounded structured-query subset, not a mutation language. Schema/conformance/publication already exist. |
| `dal-go/dalgo4auth` | README and file inventory | Placeholder, not an identity infrastructure implementation to build upon. |
| `ingitdb/ingitdb-go` | `ingitdb/config/root_config.go`, `ingitdb/collection_def.go`, README, feature inventory | Schema/configuration/validation package; no runtime DALgo policy ownership found in this inspected surface. |
| `ingitdb/dalgo2ingitdb` | `database.go`, `tx_readwrite.go`, `query.go`, `filelock.go`, `git_commit.go`, CRUD/conformance/file-lock tests | Actual current local adapter. CRUD and query, shared/exclusive cooperating-writer lock, rollback journal, optional Git commit. No `access.SecureDB`/policy loading found. Constructor returns `dal.DB`, useful for secured-handle evolution. |
| `ingitdb/ingitdb-cli` | AGENTS, `server/README.md`, `docs/adr/0001-remove-serve-command.md` reference | Server/API/auth/MCP gateway deliberately removed; old AGENTS package references are stale. Do not resurrect server as a prerequisite. |
| `ingitdb/ingitdb-specs` | `docs/schema/root-config.md`, GitHub app/PR permission specs, proposal inventory | Repository/PR permissions and GitHub credentials are distinct from per-record runtime ACL. Some docs still describe removed server/MCP functionality. |
| `openvaultdb/openvaultdb` | `spec/api/auth.md`, `spec/decisions/0004-sneat-co-identity-and-space-principals.md` | Approved hosted identity uses verified shared Sneat Firebase UID directly; current membership resolution and separate application capabilities are required. General auth spec also includes unimplemented aspirations. |
| `openvaultdb/openvaultdb-go` | `pkg/auth/{auth.go,middleware.go,store.go}` and tests; `pkg/server/{dtql.go,records.go,httperr.go,connect.go}`; `pkg/core/{core.go,query.go}`; `pkg/mount/mount.go`; README, architecture/threat-model docs | Real HTTP bearer capability auth, hashed token persistence, owner bypass, collection grants, `/query`, `/dtql`, record PATCH and batch writes. Mounts DALgo InGitDB/SQL/Firestore drivers. No row/field ACL composition at mount found. Current errors contain code/message and raw errors, no structured ACL envelope. |
| `openvaultdb-org/openvaultdb-go` | origin, SHA, go.mod, matching server/auth files | Older clone of the **same** `github.com/openvaultdb/openvaultdb-go` origin. Do not treat it as a separate service or implement twice. Current chosen clone pins DALgo v0.64.4; older clone pins v0.62.10. |
| `dal-go/dalgo2openvaultdb` | driver and driver tests, README contract | HTTP adapter and buffered writes, not a remote transaction that makes pre-image reads atomic. Must add a revision precondition for upper-layer conditional writes. |
| `datatug/datatug-cli` | `pkg/accesspolicies/{dir.go,run.go,explain.go,fields.go,accesspolicies_test.go}`, `apps/datatugapp/commands/cmd_query.go` | Already loads YAML/JSON from `~/.datatug/policies` or flags; `DecodePolicy`; principal/variable options; secured read session and explanations. No policies errors unless explicit `--no-policies`. Strong reuse candidate, not a greenfield ACL UI backend. |
| `datatug/datatug-cli` server | `pkg/server/endpoints/{routes.go,execute_endpoints.go}`, `pkg/api/execute_select_api.go`, `spec/features/serve-brokered-query-builder/README.md` | Existing HTTP query execution and read-only builder specification. Legacy select handler uses `context.Background()`, dropping request cancellation/principal context. CLI principal flags are trusted local simulation, not HTTP authentication. |
| `datatug/datatug-apps` | AGENTS, package.json, `libs/datatug/main/src/lib/queries/query-context-sql.service.ts`, `.../services/unsorted/recordset.service.ts`, related tests | Angular/Ionic/Nx, shared Sneat auth/Firebase dependencies, query UI and SQL convenience parser. Empty RecordsetService and incomplete helpers caution against assuming a full edit pipeline. Client SQL parsing is not security analysis. New UI should follow signals/OnPush conventions. |

## Exact DALgo semantics to preserve

`access/operation.go` separates `get`, `exists`, `query`, `insert`, `set`, `update`, `delete`, reserved `truncate`. `read/write/readwrite` expand to groups; write does not imply read.

`access/policy.go:matchingRules` sorts by path depth, literal specificity, restrictive effect, then rule name. A deeper allow can reopen a parent deny **inside one policy**. Independent database, bound-context and request policies intersect. `PrincipalPolicySet` unions the rule sets selected by user, roles, groups and everyone into **one** policy, then joins that intersection. Role bindings do not each become independent default-deny policies.

`access/document.go` uses `apiVersion: dalgo.io/access/v1`, `kind: AccessPolicy`, `metadata.name`, `default`, and either `scopes` or `ruleSets` plus `bindings`. Conditions already use DTQL expression shapes. Only allow rules support `where/check/fields`. `check` defaults to `where`. Read row alternatives become a disjunction; writes choose the first applicable alternative, with post-image and field checks. Query field sets conservatively intersect alternatives; this is stricter than per-row field selection. Custom `Policy` values already exist; some cannot be serialized.

`access/context.go:authorizeRequest`, `AccessPolicy.Decide`, `condition.go:matchResiduals`, and `write.go:enforceWrites` stop at the first failure. `Decision.Allowed=true` may mean **residual obligations remain**, not completed authorization. `DeniedError` is singular with English explanation and source text. The existing error must never be copied straight into a public API.

`access/condition.go:rewriteQuery` ANDs base-table predicates before execution/paging and refuses conditional joins. The wrapper's internal write pre-image read needs a `dal.ReadSession`, not necessarily a caller's public get permission; nested secured remote wrappers complicate that distinction. `access/session.go` wraps query readers for field redaction; a schema-complete post-filter verification guarantee is not established merely by the docs.

## Important gaps and discrepancies

1. DALgo docs still call predicates/projections future constraints although code and tests implement them. Feature status/prose also trails code (for example “nothing evaluates ... today”). Correct documentation in the owner work package.
2. DataTug already supplies hidden-field query-reference checks and rejects aliases under field restrictions. These checks belong in reusable DALgo enforcement for the protected profile; duplicating them in every UI would leave bypasses.
3. DataTug Explain derives attribution by splitting English `Explanation` on ` via ` and prints supplied variable values. Replace with structured provenance and disclosure projection, preserving the existing local CLI compatibility separately.
4. OpenVaultDB README/threat model contains both “no auth” and later implemented auth sections; comments in DTQL handlers still say future authentication. `/dtql` currently requires an unscoped read capability because target analysis happens later in core.
5. No discovered owner CRUD/discovery/Explain protocol, complete blockers, stable layer manifests, or safe remote pre-image revision contract. These are enabling work, not solved by simply stacking `SecureDB`.
6. Owner-token full capability access currently has no data ACL beneath it. New ACL-enabled mounts must explicitly prevent this token bypassing database policies.
7. Direct filesystem/Git access and unwrapped adapter handles bypass in-process controls. This is a threat-boundary fact, not something a policy document can repair.

## Validation and confidence

Executed `wb run -- go test ./access ./dtql ./condeval` in DALgo: **all passed** (access 0.018s, dtql 0.180s, condeval 0.003s; runner 11s). Initial sandbox attempt failed on the Go cache and was retried with approved access. Existing named tests inspected include `TestPrincipalPolicySetIntersection`, `TestConditionalQueriesThroughStub`, `TestConditionalWriteNeedsReadCapability`, `TestFieldsThroughStub`, `TestConditionalWritesThroughStub`, and codec/round-trip tests. Passing these is baseline evidence, not evidence that the proposed MVP works.

Nx inventory initially could not reach npm; local source was inspected without installing dependencies. No frontend build or cross-repo E2E was run in this design phase. No baseline claims are made for unexecuted suites. Implementation must re-pin current provider/consumer versions and run compatibility tests before using these source-level conclusions as release evidence.
