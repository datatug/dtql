# OpenVaultDB integration

OpenVaultDB is the shared remote server for InGitDB and other sources. It owns remote authentication, mounted source routing, transport and additional policies. It never replaces InGitDB's own policy ownership. DataTug can promote/open an OpenVaultDB connection for an InGitDB database without a separate InGitDB server installation.

## Mount and operation path

Extend manifest ACL configuration with explicit mode, owner policy store reference, realm, and required lower-provider capability descriptor. `pkg/mount/mount.go` opens InGitDB through its secured public constructor, then attaches OpenVaultDB's own policy layer before `core.Open` exposes operations. Existing actor capability checks remain an independent restriction. Review every escape route: `/query`, `/dtql`, GET/HEAD, PATCH/PUT/POST/DELETE, batch, schema/collection operations and direct core calls. ACL-enabled unsupported operations fail closed rather than using an unwrapped path.

Keep transport handling in `pkg/server`, source-independent execution in `pkg/core`, and source opening in `pkg/mount`. `core` accepts verified context and normalized descriptors; no separate handwritten ACL algorithm. `/dtql` should deserialize safely before collection-capability evaluation so collection-scoped grants can be assessed correctly, reusing one parsed AST. Limit bodies with explicit over-limit detection and validate query references. Avoid rendering a protected structured query into unrestricted SQL.

Static OpenVaultDB checks, InGitDB assessments and final enforcement use the reusable DALgo machinery. Expose discovery, policy read/CRUD and evaluate routes from C2. Advertise lower owner descriptors and capabilities without flattening them. Authenticate/authorize management at the actual owner; `layerId` resolves through a configured registry, not user-provided URL/file paths.

## Stores and publication

OpenVaultDB's policy files live in its own configured metadata directory beside the manifest, distinct from InGitDB metadata. Use a manifest-listed set, atomic replacement, whole-set validation, strong revision/ETag and owner administration. Changes persist across process restart and mount reconnection. Synchronize activation with the short-operation policy snapshot barrier. A DELETE of the final mandatory policy is rejected until an authorized replacement set exists; deleting a file is never ACL disablement.

The current owner-token shortcut only grants OpenVaultDB control-plane capabilities. On an ACL-enabled mount it must not skip OpenVaultDB/InGitDB data policies. Bootstrap can manage its own policy store but cannot silently edit a lower owner's policies. Map fixture owner identities to explicit separate management grants. Existing auth-disabled behavior is allowed only on legacy disabled mounts; enabling the protected ACL profile with unresolved identity configuration must reject startup/mount.

## HTTP driver and revisions

Extend `dal-go/dalgo2openvaultdb` typed error parsing to preserve C2 decision envelopes, not only code/message strings. Existing consumer code receives a typed authorization error compatible with DALgo denial matching. Include revision metadata on authorized point-read responses and accept `ifDataRevision` on key-targeted mutation/batch operation. Require backend whole-record revision support for remote conditional writes; validate within the same local short transaction as pre/post-image checks and mutation. No optimistic preflight “allow” is accepted instead.

The DataTug application can request lower dry run even when its own policy denies. `/evaluate` never calls the write executor: plan mode stays metadata-only; inspect uses bounded internal point evidence; sample reads authorized candidates under its separate bounds. Capabilities/disclosure are checked before returning enhanced diagnostics. Policy text access is not required for lower-layer enforcement; a backend may provide only an opaque decision.

## Arbitrary underlying sources

MVP reference is local InGitDB; add a focused SQLite conformance smoke proving OpenVaultDB-owned table/field enforcement over a source without DTQL-native policies. This is enabling validation, not a second full UI E2E. A legacy source can declare `nativeAcl:opaque`: OpenVaultDB adds its policies but cannot enumerate native rules or promise full lower-blocker completeness. Adapter/native denial remains authoritative and appears with opaque source provenance. A native DB being DTQL-unaware does not mean its ACL is disabled.

Capabilities must distinguish structured query restriction, field/reference checking, transactional pre/post images, whole-record revision preconditions, policy discovery/read/edit, plan/inspect/sample and diagnostic disclosure. Unsupported row enforcement, collation mismatch, hidden evidence or transactional behavior blocks that operation. Do not imply that adding a DALgo wrapper retrofits atomic authorization into every adapter. Remote custom ACL services, native policy import and GitHub-backed policy editing remain follow-up.

## Tests

Extend auth tests for actor/subject conjunction, owner-token non-bypass and current-membership changes. Server tests verify response codes, multiple blockers, redaction and CRUD preconditions. Mount tests prove lower-policy loading survives direct access and reconnect. Core tests cover every operation path and no schema/trigger side effect on denial. Driver tests preserve typed errors and revision conflicts. Native/unknown sources report partial/opaque coverage. Update stale auth/threat-model/DTQL comments as part of integration.

## Final plan precision

Policy administration and diagnostic grants are computed by an **owner-configured authority** at each layer, never copied from an upper capability boolean. InGitDB uses a root-configured grant resolver keyed by verified realm/kind/ID and registered actor, provisioned with the fixture's independent owner grants; it may read a local sidecar configuration but is not a new authentication/token service. The public trusted context carries identity/delegation facts; lower owners ignore upper policyAdmin/diagnostics booleans as lower authority. Hostile Go code remains outside the process boundary. WP4a owns this resolver binding and T09/V06 test a real upper admin request, not a forged trusted Go object.

WP5b owns the arbitrary-source capability matrix. Descriptor booleans derive from registered proven adapter capabilities and default false. The SQLite smoke requires table/field/route checks and one real bounded read; it need not claim mutation evidence/CAS. Proven before-paging predicate pushdown may support row reads without a write coordinator; a write with post-image/atomicity obligations requires the corresponding transaction capability. Opaque native sources retain authoritative opaque denial, partial discovery and no false complete diagnostics.

OpenVaultDB batch envelope remains host-owned but freezes this profile mapping: `{operations:[{id,action,executionClass,resource,mutation}]}` with 1–100 distinct IDs/normalized keys, per-item mutation.ifDataRevision and no query/native effects. Result `{items:[{id,dataRevision?}],authorization}` preserves request order and includes every successfully committed operation; no item is reported successful before whole-batch commit. A non-ACL revision conflict uses error.operationId for the failing item (lowest request-order conflict if several), error code data_revision_conflict, and zero successful items. ACL failure uses the combined C2 authorization result with all safely evaluable item blockers. Driver adapts its existing HTTP shape explicitly to this mapping; protected requests cannot silently fall back to the legacy batch schema.
