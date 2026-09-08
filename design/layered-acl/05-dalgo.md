# DALgo reusable implementation design

Contracts C1/C2/C4. Extend `access`, reuse `dtql` and `condeval`, and retain existing public APIs. Do not create a separate evaluator in OpenVaultDB or DataTug. Go signatures below are proposed API sketches whose data fields are governed by C1/C2, not implementation added in this phase.

## Responsibilities and package boundary

`access.Policy`, `AccessPolicy`, `PrincipalPolicySet`, `WithPrincipal`, `WithVariables`, `SecureDB`, `BindDB`, secured sessions and existing codecs remain. Add the portable document adapter/version decoder alongside legacy decoding. A validated editable document AST preserves bindings, metadata and targets; it compiles into existing rule objects. Do not round-trip solely through the flattened compiled policy and lose document structure.

Keep `dal` independent of YAML and HTTP. Keep protocol-specific JSON projection out of the storage interfaces. A small `access/contract` or `dtql/access` package can expose DTOs and converters without creating `access`↔`dtql` import cycles; choose one implementation home during WP2, with import-boundary tests. DTQL the repository owns normative fields and conformance vectors regardless of which Go package implements them. No immediate extraction into the placeholder `dtql-go` module is necessary.

## Proposed reusable interfaces

```go
// Sketch: all named DTOs have the normative fields in C1/C2.
type PolicyProvider interface {
    Describe(context.Context, Caller, DatabaseRef) (LayerDescriptor, error)
    Snapshot(context.Context, VerifiedPrincipal, DatabaseRef) (PolicySnapshot, error)
}
type PolicyReader interface {
    List(context.Context, Caller, PolicyListRequest) (PolicyPage, error)
    Read(context.Context, Caller, PolicyRef) (PolicyDocumentEnvelope, error)
}
type PolicyManager interface {
    Create(context.Context, Caller, PolicyDocument) (PolicyDocumentEnvelope, error)
    Replace(context.Context, Caller, PolicyRef, Revision, PolicyDocument) (PolicyDocumentEnvelope, error)
    Delete(context.Context, Caller, PolicyRef, Revision) error
}
type Inspector interface {
    Evaluate(context.Context, VerifiedPrincipal, AuthorizationRequest) (AuthorizationResult, error)
}
```

Interfaces are segregated: noneditable custom providers implement only snapshot/describe or inspector. `Snapshot` is an enforcement-only capability and may contain private policy objects; it must not be exposed as unauthenticated public policy download. `Caller` is authenticated requester/actor; `VerifiedPrincipal` is immutable trusted evaluation subject. Only resolver/authentication code constructs it at a trust boundary. Hosts do not deserialize these internal capabilities from browser JSON. Add adapters from existing `Policy` objects that attach host-owned provenance and declare whether inspection is side-effect-free.

`PolicySnapshot` includes stable owner/database/layer, revision, enabled state, mandatory policy objects, principal revision and evaluator capabilities. Mandatory owners are supplied by mount configuration. A provider error is a failed participant, never an empty policy slice. Built-in loaded snapshots are immutable; copy roles/groups/variables so callers cannot mutate an already-bound context. Cache compiled binding unions by snapshot revision and bounded subject-set key; invalidate on update and bound cache size (existing PrincipalPolicySet cache is unbounded).

## Evaluation and enforcement split

Add a richer internal assessment API (for example `Plan` and `EvaluateEvidence`) returning all decisions, blockers and obligations. Preserve `Decide` and `Authorize` as compatibility projections. Existing `Allowed` on a `Decision` must not be confused with C2 final `allowed`. Add `DeniedError` multi-decision support without breaking `errors.Is/As` or callers reading its legacy representative `Decision`; pick a deterministic representative and document it as incomplete legacy projection. Never infer structured codes by parsing English explanations.

Plan receives normalized operation descriptors, current principal snapshot, immutable policy snapshot, and the original query's field-reference provenance. It evaluates table/operation and static field policies across **all** targets. EvaluateEvidence receives private pre/post images or an authorized evidence failure. These functions are called by both secured execution wrappers and Inspector. Independent failed policies continue; inaccessible-dependent checks become explicit unevaluated obligations. Do not run all losing rules and call every one a blocker; within-policy precedence is unchanged.

Refactor `guard.authorizeRequest`, `AccessPolicy.Decide`, `matchResiduals` and write evaluation to accumulate safe independent failures before returning. Field policies that apply unconditionally can be evaluated despite another policy's row denial. Row-dependent field alternatives cannot be guessed if their deciding row is unavailable. Batch targets are evaluated before delegating any writes. Built-in custom policy callbacks that are not declared pure are enforced on execution but produce `unsupported` inspection coverage; never replay side effects to explain them.

A `QueryPlan` holds original caller references separately from owner-injected predicates. Move DataTug's hidden-reference/alias safeguards into this reusable path. Deny filters/sorts/projections on forbidden fields. Apply all row filters before paging. Field redaction remains defense in depth, not a substitute for query semantics. An adapter returning inconsistent rows must be rejected before publishing a partial success result; the supported bounded MVP API buffers up to its row limit. Streaming late-error guarantees are follow-up.

Private mutation evidence must include whole-record revision and final candidate image. For `set`, compute deleted as well as supplied field paths; do not let replacement remove a protected field. For nested object set, include all affected descendants; a grant to one child cannot replace its siblings. Deny unsupported updates rather than approximate post-images. Strengthen wrappers without requiring caller-visible read permission for in-process enforcement reads. Across HTTP, require advertised revision capability or fail closed for row-dependent writes.

## Tests and migration gates

Keep baseline legacy tests passing. New tests: golden DTQL documents and canonicalization; strict duplicate/unknown-key parsing; user+role+group union; tie precedence; differential legacy adapter conversion; multiple-policy/target blocker sets; read plan conditional vs inspect exact; unsupported custom inspection; safe codes without English dependencies; no hidden-field filter/sort bypass; set deletion/nested writes; immutable principal snapshots; missing enabled policies; adapter residual rejection; schema/DTO import boundaries.

Use recording adapters to prove no read in plan mode, only bounded reads in inspect mode, and zero writes/triggers/commits in both modes. Property tests for cross-policy monotonicity and result invariance under policy iteration order are high-value. Fuzz path/field parsing, duplicates and depth/size bounds. Existing `end2end/test_access.go` is the extension point for adapter conformance. Do not build a new fleet-wide ACL test framework.
