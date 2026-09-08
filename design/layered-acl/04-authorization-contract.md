# DTQL authorization, discovery and dry-run wire contract

Contract C2; proposed `dtql.org/authorization/v1`. This is the normative field/type schema for implementers. JSON Schema artifacts and generated Go/TypeScript bindings are WP1/WP2 deliverables; this document does not claim a generated schema already exists. Existing query YAML remains unchanged. Management and decisions use JSON over HTTP; YAML remains the policy editing representation.

## Request schema

An `AuthorizationRequest` has the following closed-object shape. Unknown fields, duplicate keys and mismatched discriminators reject before evaluation. The same normalized `operation` descriptor feeds dry run and execution; no separately interpreted diagnostic query language.

| Field | Type | Requirement |
|---|---|---|
| `apiVersion` | string | Exact `dtql.org/authorization/v1`. |
| `mode` | enum | `plan`, `inspect` or `sample`; dry run only, never mutation. |
| `sample` | object | Required only in sample mode: `{query:{format,text,parameters?},limit:integer}`; limit 1–100. One table, one operation template; see sampling rules below. |
| `subject` | object | Optional `{realm,id,kind}` for privileged other-principal Explain; omitted means authenticated self. `kind`: `user`, `service`, `application`, `agent`. |
| `simulation` | object | Optional `{roles:[id],groups:[id],attributes:{name:scalar}}`, only with `access:simulate`; marks hypothetical result. Cannot be sent as execution authority. |
| `operations` | array | 1–100 operation descriptors with distinct caller-provided `id` (≤128 bytes). |
| `diagnosticLevel` | enum | Requested `ordinary`, `references`, `policy`; server attenuates independently at every owner. |

`plan` MUST NOT execute a data query or read stored rows. It may read policy/config/schema/identity metadata. It returns static blockers plus residual row obligations, and `conditional` where row evidence is necessary. `inspect` MAY perform bounded point reads inside the owner evaluator for supplied concrete keys; no mutation, trigger, external effect, result-set scan, or user query execution. It does not return the inspected row. Inspect requires self-operation inspection or authorized Explain scope, and disclosure limits still apply. Discovery of policies is independent of either mode: an evaluator can return a safe denial without exposing its policy text.

With no concrete rows requested, plan returns disclosable **admission conditions** and field/post-image restrictions rather than guessing which rows fail. Each restriction includes `representation:expression|reference`, optional structured `expression` in C1 condition shape, and optional `omissionReason:private|too_large|unsupported`. An omitted expression retains an opaque restriction ID and disclosable policy/layer reference. Do not synthesize a negated predicate or expand combined predicates into exponential normal form. Combine by explicit references: `allOf:[restrictionId,...]` at the operation level (optional alongside `restrictionIds`); each policy retains its own bounded expression. Limit a disclosed expression to 16 KiB/256 nodes and the complete response to 256 KiB; above that emit `too_large` references, not silently truncated predicates. This changes representation, not internal evaluation completeness. Private expressions produce redacted disclosure; size omission does not mean evaluation failed. Never substitute a caller-readable simplified condition that is weaker than actual enforcement.

### Bounded top-N sampling (user direction)

`sample` selects at most N rows using the supplied supported DTQL query, then evaluates the single operation template for those keys; this is deliberately a read-producing **internal** diagnostic operation, unlike plan mode. No record values are returned and no mutation is executed. Default order is canonical record ID ascending; an explicit supported ordering adds record ID as a final tie-breaker. Apply authorization filters before ordering/limit. Limit bounds inspected records, and the source must also honor the elapsed/resource budget; N alone is not a scan-cost bound. Stop with partial coverage on budget exhaustion.

Ordinary self-sampling is confined to the intersection of subject-readable rows and requester-readable rows through every participating layer. Other-principal Explain does not grant the requester that principal's data visibility. Broader diagnostic sampling requires separate owner-granted `access:inspect-protected`; it remains bounded and redacted, and is not required in MVP. Never retrieve arbitrary hidden rows under a service credential merely to fill N. For an intended update/delete, sample the visible candidate rows **before applying that operation's write ACL**, otherwise the sample would omit precisely the denied rows being investigated. For insert, sample mode is invalid; inspect evaluates supplied candidate data without a stored-row read.

Result adds optional `sample:{requestedLimit,evaluatedCount,selection:"readable_candidates",order,exhaustive:false}`. Counts refer only to the authorized sampled set, not hidden or total matching rows. Sampled per-row decisions use only disclosable keys. The overall result describes **that sample** and must never be shown as permission for a whole bulk operation; `exhaustive` is always false in MVP, even if fewer than N rows return. A sample with zero rows is not affirmative data access. Use `result:conditional`, `allowed:false` with no sampled per-row decisions. Mutating execution never accepts sample results as authority.

An operation descriptor is `{id, action, resource, query?, mutation?}`. `action` is one leaf operation from C1. `resource` is `{databaseId,path,table?,rowId?,columns?}`: path is normalized logical path, table an optional derived display identifier, rowId an optional string key, columns an array of concrete field segment arrays. For a table path and query, no rowId. Resource names must agree with the parsed query/mutation and server mount mapping; server derives rather than trusting redundant fields. Real request fields must not claim policy owner IDs.

`query` for `query` action is `{format:"dtql-yaml",text:string,parameters?:{name:scalar}}`. The text uses the existing DTQL single-source query shape. Mutations are discriminated: insert/set `{data:object}`; update `{changes:[{op:"set",path:[segment,...],value:JSONValue}]}`; delete `{}`. All mutations include optional `ifDataRevision:string` for execution, mandatory when an upper conditional write relied on a remote pre-image. Other update operators are rejected in MVP; architecture and existing DALgo model remain extensible. Body is bounded to 1 MiB; report oversized before parsing rather than silently truncating at a byte limit.

Public users cannot provide `effectivePrincipal`, `policySet`, `owner`, trusted variables, a signed-result substitute, or mode `execute` to this endpoint. The caller's actor and transport authority always come from authentication. Execution uses existing read/PATCH/batch endpoints with the same descriptor normalizer.

## Result schema

`AuthorizationResult` is a closed object with required `apiVersion`, `requestId`, `result`, `allowed`, `hypothetical`, `operations`, `layers`, `blockers`, `coverage` and `restrictions`. `requestId` is a server-generated opaque correlation ID.

| Field | Type / semantics |
|---|---|
| `result` | `allow`, `conditional`, `deny`, `indeterminate`. |
| `allowed` | Boolean; true **only** for final complete `allow`. Never true for residual obligations. |
| `hypothetical` | Boolean; true for supplied simulation identities/memberships/attributes. |
| `operations` | Array of `{id,action,resource,result,restrictionIds:[id]}` matching requested operation IDs. |
| `layers` | Array of layer decisions described below, including known unavailable/disabled participants when visible. |
| `blockers` | Array of structured blocking facts; may be redacted/coalesced. Empty does not imply allow. |
| `coverage` | `{evaluation:complete|partial,disclosure:full|redacted,truncated:boolean,unevaluated:[{operationId,layerId?,reason}]}`. Reason enum: `row_evidence_required`, `evidence_not_authorized`, `source_unavailable`, `unsupported`, `budget_exceeded`. |
| `restrictions` | Array of `{id,operationId,layerId?,kind:row_filter|field_allowlist|post_image_check|opaque,policyRef?,expression?,fields?,enforced:boolean}`. Resolved secret values/predicate text omitted unless explicitly disclosable. |

Layer decision: `{layerId,source,aclState,result,policyRevision?,decisions:[...]}`. `aclState`: `enabled`, `disabled`, `unavailable`. `source`: `{ownerId,provider,databaseId,kind,reference?}`. `kind`: `datatug`, `openvaultdb`, `ingitdb`, `application`, `adapter`. `layerId` is a configured stable instance ID, not merely the product name; two OpenVaultDB hops have different IDs. `reference` is an opaque admin-safe locator, never an arbitrary URL the client should fetch.

Each participating decision is `{operationId,result,policyRef?,scope,ruleRefs?:[id],bindingRefs?:[{kind,id}],restrictionIds:[id]}`. `scope`: `database`, `table`, `row`, `column`, `operation`, `principal`, `configuration`. Policy reference is `{ownerId,databaseId,policyId,revision,ruleId?}`. Policies not visible to the caller can be replaced by opaque layer-level decisions. Always preserve internal attribution before redaction. Provider-supplied source names are anchored to the configured source; cannot impersonate an upstream owner.

Each blocker is `{operationId,code,scope,resource?,layerId?,policyRef?,slot?,columns?,retryable?}`. `slot` is `where`, `check`, `fields` or absent. No required English `message` or `reason` string. Clients decode the code and structured facts for localization. A condition literal, resolved param, raw exception, host file path or hidden row identifier MUST NOT be emitted merely because it exists in a DALgo error.

Reduction: any definitive policy denial → `deny`; otherwise any failed/unevaluated required participant → `indeterminate`; otherwise outstanding enforceable row/post-image obligations → `conditional`; otherwise `allow`. `plan` can be conditional with `evaluation:complete` because all plan-stage work finished; every outstanding row obligation must still be explicit. `inspect` with unavailable evidence is partial/indeterminate unless another denial already decides the result. An execution success has `allow` and every required obligation enforced. Hidden diagnostics can have `disclosure:redacted` with complete internal evaluation. Never expose hidden blocker counts through a count field.

Deduplicate by `(operationId,layerId,policyId,revision,ruleId,code,scope,slot,canonical resource,column)`; sort operation IDs by request order, then layer path order, policy ID, rule ID, code and path. Deduplication never merges two owners. To bound payload, group multiple columns for the same rule/code into sorted `columns`. Evaluate up to 100 operations, 32 concrete columns per operation, 100 policies per owner, 1,000 rules per policy, predicate depth 16, and 1,000 emitted facts. Configuration exceeding supported policy limits is rejected at load. Request excess rejects 400; runtime budget of 2 seconds for dry run yields explicit partial coverage. Cancellation stops work. No unbounded fan-out or enumeration; truncation makes completeness partial and never grants execution.

## Reason-code taxonomy

Use orthogonal code + action + scope instead of an expanding `TABLE_UPDATE_DENIED`/`ROW_DELETE_DENIED` Cartesian product. The existing OpenVaultDB outer lowercase error codes remain separate from these DTQL codes.

| Code | Meaning and required facts |
|---|---|
| `ACCESS_DENIED` | Safe coalesced denial when details cannot be disclosed; no policy/row existence implied. |
| `ACL_RULE_DENIED` | Winning explicit deny; action, scope, policy/rule where visible. |
| `ACL_NO_MATCH` | Mandatory policy has no admitting rule/subject match; policy where visible. |
| `ACL_ROW_PREDICATE_FAILED` | Actual inspected pre-image fails row admission; `scope:row`, `slot:where`. |
| `ACL_POST_IMAGE_FAILED` | New row/candidate post-image fails effective check; `slot:check`. |
| `ACL_COLUMN_DENIED` | Requested/touched field outside policy allow-list; `scope:column`, `slot:fields`, safe requested columns. |
| `ACL_EVALUATION_FAILED` | Type error/unresolved trusted variable/custom evaluator failure; never raw error text. |
| `ACL_CONFIGURATION_INVALID` | Enabled owner missing/invalid/incomplete policy set. |
| `ACL_SOURCE_UNAVAILABLE` | Required policy/evaluation provider unreachable. |
| `ACL_ENFORCEMENT_UNSUPPORTED` | Adapter cannot enforce required obligations or request form. |
| `ACL_PRINCIPAL_UNRESOLVED` | Authenticated identity cannot resolve authoritative membership/context. |
| `ACL_CAPABILITY_DENIED` | Authenticated actor/token lacks required operation scope/grant. |

Syntax/version/body validation is 400 and not a pretend policy denial. Authentication failure is 401. Policy document revision conflict is 412 `policy_revision_conflict`; missing management precondition is 428. Record revision conflict is 409 `data_revision_conflict`, not an ACL blocker. These non-ACL outer errors have fixed machine codes and requestId; policy predicate errors may use `ACL_EVALUATION_FAILED` only after the request is valid.

## HTTP binding

Base: `/v1/databases/{db}/access`. In DataTug these calls are scoped by its authenticated project/connection route; OpenVaultDB exposes this path directly. DALgo provides equivalent Go provider interfaces, not an HTTP server.

| Method/path | Action | Response |
|---|---|---|
| GET `/layers` | `policies:discover` | Visible descriptors: source, ACL state, policy revision, supported modes/formats, management capabilities, required/opaque lower participants. No policy text. |
| GET `/policies?layerId=...` | `policies:list` | Authorized metadata page `{items,nextCursor?}`. Cursor opaque, owner-generated; filtering before pagination. |
| GET `/policies/{policyId}?layerId=...` | `policies:read` | `{source,policyRef,format,document,capabilities}` plus strong ETag. Document is policy AST or YAML text according to negotiated format. |
| POST `/policies?layerId=...` | `policies:create` | 201 descriptor, owner-assigned revision; duplicate stable ID is 409. |
| PUT `/policies/{id}?layerId=...` | `policies:update` | Full validated replacement with `If-Match`; same ID/target, 200 new descriptor/ETag. |
| DELETE `/policies/{id}?layerId=...` | `policies:delete` | Requires `If-Match`, 204; does not disable ACL or bypass last-policy guard. |
| POST `/evaluate` | Self dry run or `access:explain`; `access:simulate` if hypothetical | 200 AuthorizationResult even when result denies. Request/auth failures use ordinary HTTP errors. |

Capabilities are `{discover,list,read,create,update,delete,explain,simulate,diagnostics}` booleans computed for **the current requesting principal at the owner**, plus editor format support. They are UI hints, not durable permission tokens. Reading policy text does not imply edit or explain-as-another. Owner reauthorizes every operation. A list request without visibility may return empty/opaque topology instead of leaking policy presence. Disallowed policy-ID probes use a consistent not-found/forbidden projection.

Add `policyAdmin` to this capability object, derived from an explicit owner-scoped `policies:admin` grant. This is the only role that may see `metadata.visibility:private` policies or change visibility. Ordinary list/read/diagnostics/Explain permissions apply only to public policies; requests for policy diagnostic level are attenuated accordingly. Policy-admin classification authority does not automatically grant data access, another owner's administration, or create/update/delete actions. Owners may provision these management grants together, but checks remain distinct. A private policy's existence/count is not separately enumerated to non-admins; coalesce its restriction into a generic owner/source result only where that source is already visible. Never return a hidden policy ID in blockers, ETags or cursor contents.

Execution denial binding: HTTP 403 `{"error":{"code":"access_denied","requestId":"...","authorization":{...}}}`. Execution indeterminate/unavailable: 503 `authorization_unavailable` with result envelope, no execution; unsupported profile: 422 `authorization_unsupported`, no execution. Preserve these typed envelopes through `dalgo2openvaultdb` and DataTug. `errors.Is(err, access.ErrAccessDenied)` remains true for actual denials. Unknown **returned** future ACL codes display a generic denial with preserved code, never permit access; unknown **input** enums reject.

## Example: three safe independent blockers

```json
{
  "operationId": "u1",
  "code": "ACL_COLUMN_DENIED",
  "scope": "column",
  "layerId": "ovdb-crm",
  "policyRef": {
    "ownerId": "ovdb-local",
    "databaseId": "crm",
    "policyId": "financial-fields",
    "revision": "p7",
    "ruleId": "update-fields"
  },
  "slot": "fields",
  "columns": [["credit_limit"]]
}
```

The `blockers` array can contain this fact, an InGitDB `ACL_ROW_PREDICATE_FAILED` for the caller-supplied path, and DataTug `ACL_COLUMN_DENIED` for status. This is a blocker fragment, not a complete result example. For an ordinary caller the row failure may be replaced by `ACCESS_DENIED` without row/policy identity; the diagnostic test principal explicitly has reference disclosure privileges at every owner. No response contains row values.
