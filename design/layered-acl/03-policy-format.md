# DTQL ACL policy format and semantics — proposed v1

Contract C1. Proposed `apiVersion: dtql.org/access/v1`; this identifier is a version discriminator, not a claim that a public schema URL is already deployed. DTQL owns this portable contract; DALgo supplies its first conforming implementation. `dalgo.io/access/v1` remains accepted by existing legacy decoders. Do not silently reinterpret or rewrite legacy documents.

## Reuse and representation

Canonical editable text is UTF-8 YAML 1.2 in its JSON-compatible subset; JSON represents the same abstract document. This adopts DALgo's existing document and condition shape rather than creating a policy DSL. The first portable profile supports existing hierarchical allow/deny, row `where/check`, field **allow-lists**, and principal bindings. The [reviewed mask extension](18-mask-supplement.md) adds collection and column include/exclude sets within the same evaluator. A UI “deny column” action produces an explicit allowed-field restriction in a separate policy against the current schema; it never invents a `deny fields` rule that DALgo cannot execute. New fields stay denied by that explicit list. Predicate-bearing deny rules are unsupported in this profile; represent positive admissible rows rather than introducing negation semantics.

```yaml
apiVersion: dtql.org/access/v1
kind: AccessPolicy
metadata:
  name: tenant-access
  visibility: private
target:
  database: crm
composition: dalgo-hierarchical-v1
default: deny
ruleSets:
  editor:
    - path: /customers
      rules:
        - id: read-tenant
          effect: allow
          operations: [get, exists, query]
          where:
            op: "=="
            left: {field: tenant}
            right: {param: tenant}
        - id: update-tenant
          effect: allow
          operations: [update]
          where:
            op: "=="
            left: {field: tenant}
            right: {param: tenant}
bindings:
  users:
    user-alice: [editor]
  roles:
    crm-editor: [editor]
  groups:
    crm-team: [editor]
```

The example names an opaque user in the configured identity realm. Real deployment uses its established ID, not an OAuth subject or display name. `tenant` is a host-resolved trusted authorization attribute, never a query-supplied parameter. This policy denies operations not explicitly granted; a fixture requiring other operations supplies corresponding rules.

## Document grammar

All maps reject unknown keys. Keys are case-sensitive. Nonempty strings are bounded to 256 UTF-8 bytes unless otherwise specified. IDs use `[A-Za-z0-9][A-Za-z0-9._-]{0,127}` for policy/rule/rule-set IDs. Principal binding keys are opaque strings bounded to 256 bytes, interpreted within the owner's realm; they need not obey the policy ID grammar.

| Field | Required type and meaning |
|---|---|
| `apiVersion` | Exact string `dtql.org/access/v1`. |
| `kind` | Exact `AccessPolicy`; audit-selection policies remain separate and never authorize data. |
| `metadata` | Object with required `name`, stable policy ID; optional `description` ≤2048 bytes, non-normative author text; optional `visibility: private|public`, default public. |
| `target` | Object with required `database`: exact local owner database ID. No regex or wildcard in MVP. Mount aliases resolve server-side to the owner's database ID before evaluation. |
| `composition` | Required exact `dalgo-hierarchical-v1`; unknown value rejects. Across documents composition is always intersection. |
| `default` | Required exact `deny`. |
| `scopes` | Nonempty scope array for a policy applying to every principal; mutually exclusive with `ruleSets/bindings`. |
| `ruleSets` | Nonempty map of rule-set ID to nonempty scope array, required with `bindings`. |
| `bindings` | Object with `users`, `roles`, `groups` maps to nonempty arrays of existing rule-set IDs, and optional `everyone` array. At least one binding required. Matched sets deduplicated. |

A scope has exactly one selector `path` (string), `collectionGroup` (string), or `opaqueQuery: true`, plus nonempty `rules` and/or child `scopes`. MVP wire interchange can preserve the latter two legacy selectors, but loading either into an enabled protected owner rejects the complete candidate set as ACL_CONFIGURATION_INVALID; never ignore an unsupported scope. Representable unconditional truncate rules may load, but invoking truncate is always unsupported in the protected profile, so such rules cannot reopen an execution path. A disabled legacy source does not thereby gain the new profile's conformance claim.

A rule has required `id`, `effect` (`allow` or `deny`), `operations` (nonempty array); optional `where`, `check`, `fields`. IDs are unique within a plain policy and within each rule set; the qualified rule ID is `ruleSet/rule` for bindings. `fields` is a nonempty array of patterns; absence means all fields, and explicit `[]` rejects to avoid omission/empty serialization ambiguity. `where/check/fields` are allowed only on `allow`. `check` without `where` constrains new data but not reads. Neither `where` nor `check` accepts explicit null.

Leaf operations: `get`, `exists`, `query`, `insert`, `set`, `update`, `delete`, `truncate`. Authoring conveniences `read`, `write`, `readwrite` expand to the existing leaf sets. Conditional/field-restricted groups must canonicalize to explicit supported leaves and exclude truncate; explicit restricted truncate is invalid. New serializers emit explicit leaves in that order. Unknown operations reject. `truncate` is representable but not executable in MVP. CRUD on policy resources uses a separate administration action namespace, never these data grants.

## Paths, fields, expressions

Path selectors reuse DALgo alternating collection/record segments: `/customers`, `/customers/123`, `/spaces/{spaceID}/customers`. Scopes apply to descendants. Nested fragments append structurally; canonical output uses the same nested tree, not a semantic flattening. `*` and `{capture}` only occupy record-ID positions; `/**` is optional terminal inherited-subtree notation, not arbitrary glob syntax. Root `/` matches all logical resources. Only string record IDs are portable in this profile. IDs containing reserved separators use percent encoding decoded exactly once into structural segments; adapters must use the common normalizer, never filesystem string concatenation. Reject malformed encodings, empty segments, dot traversal, invalid wildcard placement and collisions after normalization. Preserve case; no Unicode/case-fold aliasing.

Field names are dotted paths using DALgo's existing `fields` patterns (whole-segment `*`, permitted prefix/suffix wildcard forms). A concrete request uses field segment arrays on the wire, making nested `address.city` unambiguous. Literal dots in a segment, ambiguous pattern escapes and whole-object replacements under descendant-only grants are rejected by the portable profile until lossless common escaping is specified. MVP UI offers schema field checkboxes and exact names; it preserves read-only supported patterns rather than expanding arbitrary wildcard expressions into a lossy form.

Conditions use exactly one of `{op,left,right}`, `{and:[conditions...]}`, `{or:[conditions...]}`. Groups must be nonempty. Left is `{field: name}`. RHS is `{value: scalar}`, `{values: [scalars...]}` for `In`, or `{param: name}`. Scalar: string, boolean, null, or finite number in the cross-runtime exact range; integers are limited to ±(2^53−1), with larger IDs encoded as strings. No arbitrary objects, YAML tags, timestamps or execution expressions. Comparisons are `==`, `In`, `>`, `>=`, `<`, `<=`. Missing fields do not match; explicit null only equals null, is not ordered. Numeric comparison uses numeric values without string coercion; incompatible types return a safe evaluation error. String ordering is UTF-8 lexical, not locale/collation-dependent. Adapters unable to match these semantics must reject the operation or use proven equivalent enforcement before paging.

Authorization params are `currentUser`, `principal.roles`, `principal.groups`, `now`, declared host attributes such as `tenant`, and valid `path.<capture>`. Resolve each once from a trusted snapshot; a missing variable fails closed. `now` is a finite integer Unix timestamp in milliseconds in the exact numeric range, read once from the host clock per layer; it is not a client timestamp. Policies comparing timestamps must use that numeric representation; string/date coercion is forbidden. principal.roles/principal.groups and other list-valued params are valid only as In RHS; other operators return ACL_EVALUATION_FAILED. Scalar In RHS is a type error. No filesystem/env/network expressions. The MVP cross-layer fixture uses no time-dependent expressions. Query parameters have a distinct untrusted namespace and cannot override these variables.

## Within-policy evaluation algorithm

1. Resolve the verified subject in the owner realm. `users` matches only kind `user`; service/application/agent subjects match only explicitly assigned roles/groups or `everyone`. `currentUser` is the stable ID for human users only; use with another kind is an unresolved-variable error. Select `everyone` and matching user/role/group rule sets; union and deduplicate them. No match leaves a default-deny policy. Prefix rule IDs by rule-set ID and sort deterministically; declaration/map order is not authority.
2. Select rules matching operation and structural resource. Sort descending path depth, descending literal specificity, restrictive effect first, then ascending qualified rule ID.
3. Unconditional rules decide the remaining space. Conditional allow rules ahead of that terminal authorize only rows meeting their predicates. A deeper allow may reopen a less-specific deny; an equally specific deny precedes allow.
4. For read row eligibility, conditional alternatives are ORed, bounded by the first unconditional rule. An unconditional allow makes row eligibility unrestricted, though field restrictions remain. Query field bounds conservatively intersect all possible deciding allow alternatives in the current DALgo profile.
5. For an existing-row write choose the first alternative whose pre-image `where` is true, otherwise terminal allow; do not fall through to a later allow after that chosen rule's field or post-image check fails. For a new row choose the first alternative whose effective check succeeds, then check fields. `check` defaults to `where`. No admitting alternative means deny.
6. Intersect independently applied policies. Conjoin their row obligations and intersect their field sets. A denial in one policy is never reopened by another.

Policy IDs and rule IDs affect stable attribution; rule ID also currently breaks equivalent-priority ties. The editor must warn that renaming a rule can change precedence. MVP does not add policy renaming or arbitrary rule-order controls. Migration tests must prove equal decisions for legacy supported constructs, including this tie behavior.

## Provenance and round-trip editing

Policy visibility is owner-enforced classification. `private` hides policy identity, text, rules, predicates and policy-specific references from every caller except an explicitly authorized **policy admin at that owner**. Discovery/read/Explain/diagnostics privileges alone cannot reveal a private policy. `public` merely permits ordinary owner authorization checks to disclose it; it is not public unauthenticated access. Create/import defaults public. PUT omission preserves the stored visibility; canonical serialization always emits the effective visibility explicitly. A non-admin cannot change classification by omitting it. Changing this classification requires owner policy-admin authority, independently of ordinary edit permission. The owner persists and attests the effective classification and rejects a non-admin replacement that changes it. Legacy imports default public. Private restrictions may still return opaque request-scoped restriction IDs/generic denial without revealing hidden policy count or identity. These IDs cannot be dereferenced by non-admins. Classification applies to cached responses and audit access as well as policy download.

Source, owner, layer, resource mount, Git revision and editability are returned in an **owner-attested envelope**, not trusted from user-editable text. `metadata.name` identifies the policy locally; global identity is `(ownerId, databaseId, policyId)`. A fetched descriptor includes the revision/ETag and source reference. Export may bundle that envelope beside the document, but import must discard claimed authority and obtain provenance from the new owner.

MVP promises semantic round-trip and canonical idempotence: parse → document AST → edit → validate → serialize → parse preserves all supported fields/meaning; a second canonical serialization is byte-identical. It does not preserve comments, original whitespace, quoting, anchors or source ordering. Reject anchors/aliases, duplicate keys, merge keys, custom tags, multiple YAML documents and duplicate JSON object keys. This is stronger than assuming generic decoder unknown-field rejection suffices. Normalize map keys lexically, operation arrays to leaf order, subject/rule-set references as sorted sets; keep semantic scope/rule structures and qualified IDs intact. The UI explains that saving canonicalizes formatting. Raw text editing is optional, but parse/serialize APIs support it.

Unknown-version, custom or nonportable policy documents remain represented by read-only descriptors. Never edit through a reduced AST and drop unsupported fields. A supported document with complex scope/pattern structure outside the form editor's fidelity can likewise be read-only or edited through the optional raw editor. Legacy conversion is an explicit owner-admin action with before/after differential fixtures; do not strip `target`/`composition` and call that a complete migration.

## Execution routes and inclusion/exclusion masks — user addition

C7 adds optional `execution` to the same AccessPolicy document; it is a conjunctive gate evaluated by DALgo before route dispatch, not a competing policy engine. It is independent of table/row/field admission. Omission adds no route restriction (legacy meaning preserved); a present object defaults to deny for every unlisted class. `execution.allow` is an array, possibly empty to forbid every class. Each entry has `class:dtql|native_sql|native_graphql|stored_procedure`. dtql/native_sql/native_graphql entries contain only class. stored_procedure entries additionally require an exact owner-resolved `namespace`, nonempty `include` mask array and optional `exclude` mask array. Multiple entries union **inside this one gate**; exclusions for the same class/namespace across entries are accumulated and win over every inclusion. Independent policies/layers still intersect, so another owner's deny cannot be reopened.

Example route restriction (inside an otherwise applicable policy):

```yaml
execution:
  allow:
    - class: dtql
    - class: stored_procedure
      namespace: public
      include: ["User_*"]
```

This forbids native SQL/GraphQL and every stored procedure outside public.User_*. To allow all procedures in public except sys_*, use `include: ["*"]` and `exclude: ["sys_*"]`. To forbid all stored procedures while retaining DTQL, use only `{class: dtql}`. This is a default-deny allow-list, not an explicit deny-all rule that magically loses to an exception. Admin authors a mandatory narrowing policy with applicable ordinary action scopes plus this route gate; a present gate constrains all operations covered by that policy and does not bypass its ordinary default deny.

Masks match **canonical callable base names**, not query text, argument values, arbitrary paths or schema-qualified SQL strings. Namespace is matched separately and exactly. MVP grammar: anchored, case-sensitive Unicode literal characters with the bounded nameMask grammar in 18 (literal, exactly *, one leading * or one trailing *); no regex, `?`, character classes, recursive globs or escape syntax. Reject names/masks with path separators, dots, controls or ambiguous literal `*`; masks ≤128 characters, ≤32 masks per entry, ≤32 entries. The adapter resolves source case folding/quoted identifiers to canonical namespace/name **before** matching; policy author sees those canonical names. If lossless canonical resolution is unavailable, fail closed. Overloads share the same name-level policy in v1; signature-specific permissions are future work. This portable namespace profile may reject source-specific names rather than guess.

`executionClass` is required in normalized operation descriptors. Existing DTQL requests normalize to dtql; wire A2 requests specify it explicitly. Native SQL/GraphQL and procedure invocation are separate classifications, not new CRUD operations. Stored procedures may read/write arbitrary resources; class/mask permission is necessary but never sufficient for effects authorization. Server chooses the class from the actual ingress/parsed operation, and rejects a conflicting caller label. SQL `CALL`, SELECT invoking side-effecting functions, GraphQL resolvers and multi-statement SQL must not bypass procedure restrictions through a native-query label. Where effects/call graph cannot be bounded and enforced, reject the entire route.

**MVP proves actual DTQL route gating and serializes/evaluates callable masks with conformance fixtures. It does not enable protected native SQL, GraphQL or stored-procedure execution.** These routes remain unsupported even if a route gate would permit their names. Providers must advertise that limitation; native effects analysis and callable execution are follow-up. A plan can return a static route denial without a call; a matching mask does not produce overall allow when effect enforcement is unsupported. DTQL table/path selectors retain their existing structural grammar; this addition does not silently turn them into arbitrary globs.
