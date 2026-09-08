# A2-S1 — collection and column inclusion/exclusion masks

User direction after the A2 main packet froze: inclusion/exclusion patterns also apply to collections and columns. Both independent reviewers receive this identical supplement after sealing their main review; it is reviewed before reconciliation. This extends C1/C2/C7 and the package/test plan, not native execution scope. Native SQL/GraphQL/procedure effects remain unsupported.

## Reuse verified against existing DALgo

At audited DALgo commit `6b0be09b95bd3afc90309763671791e17264dffe`, `access/fields.go` already parses per-segment literal, `*`, prefix* and *suffix masks, supports dotted paths/parent descendants, and intersects `fieldSets`. It has no exclusion set. Extend that same parsed representation with exclusions and reuse its segment matcher; preserve legacy `fields` behavior. Do not expand masks into a schema snapshot or add a competing evaluator.

All new name masks (procedure base names, collections, individual column segments) use the same bounded existing segment grammar: literal, exactly `*`, one trailing `*` or one leading `*`; reject interior/multiple stars, `?`, regex, escapes, control characters and separators. Matching is anchored/case-sensitive after source canonicalization. This **tightens** A2's more permissive procedure-star wording before freeze; User_* and sys_* are unchanged. Maximum 32 include and 32 exclude patterns, each ≤128 UTF-8 bytes. Include must be nonempty; exclude may be omitted/empty. Defaults are never inferred from exclude alone: use explicit include:["*"]. A separate ordinary deny can deny all; no ambiguous empty include meaning unrestricted.

## Collection mask

Add optional `collectionMask:{include:[nameMask,...],exclude?:[nameMask,...]}` to AccessPolicy. It is a **mandatory conjunctive admission gate**, like execution, not a wildcard path with invented specificity. It applies to every data operation assessed by that policy, irrespective of which bound rule allows it. Omission adds no collection gate. When present, the normalized owner collection must match an inclusion and no exclusion. Exclusion wins inside the gate. No match is `ACL_COLLECTION_DENIED` with table scope; another independent policy or owner cannot override it.

MVP targets single root collections. Match canonical collection base name resolved from the parsed query/key, not caller display aliases; preserve the owner's database identity. Existing `path`/rule hierarchy still decides ordinary operations, row/field restrictions and bindings. The collection gate cannot grant an ordinary action that the policy denies. Nested/collection-group/native requests whose touched collections cannot be fully resolved are unsupported under this profile, never treated as matching `*`.

Example complete narrowing policy (applicable to all authenticated subjects already admitted by other mandatory policies):

```yaml
apiVersion: dtql.org/access/v1
kind: AccessPolicy
metadata:
  name: public-customer-surface
  visibility: public
target:
  database: crm
composition: dalgo-hierarchical-v1
default: deny
collectionMask:
  include: ["*"]
  exclude: ["sys_*"]
execution:
  allow:
    - class: dtql
scopes:
  - path: /
    rules:
      - id: surface
        effect: allow
        operations: [get, exists, query, insert, set, update, delete]
        fieldMask:
          include: ["*"]
          exclude: ["secret_*", "address.internal_*"]
```

This narrows access to non-sys_ root collections and permitted columns. It does not replace the required tenant/country/workflow policies or grant the actions they deny. For only User_* collections, use collectionMask include:["User_*"]. Schema discovery applies this gate before collection enumeration/pagination. Public policy text intentionally discloses its mask literals subject to ordinary policy read authority.

## Column mask

Add optional `fieldMask:{include:[fieldPattern,...],exclude?:[fieldPattern,...]}` to an **allow** rule, mutually exclusive with legacy-compatible `fields`. Deny rules forbid both. `fields:[...]` retains exact existing semantics and is not silently reserialized as fieldMask. A fieldMask parses to one internal fieldSet containing inclusion and exclusion patterns. `allows(path) = any(include matches path) AND NOT any(exclude matches path)`. Excluded parent paths exclude every descendant; including a parent covers descendants subject to exclusions. A literal dotted field pattern always means nested segments, never a literal dot in a physical key. Reject ambiguous physical keys at protected normalization/evidence boundaries as specified in C1.

Column patterns reuse current DALgo dotted segment grammar and parent/subtree semantics: `address.*` selects descendants and current parent-prefix behavior; `address.internal_*` excludes matching nested fields and their descendants. The implementation must not return an object wholesale just because its parent is included: recursively redact excluded descendants. Setting/replacing a parent validates every affected descendant, including removed fields; an excluded child blocks whole-object replacement even if its value is unchanged when the operation touches it. Arrays and complex provider paths are unsupported unless existing conformance proves complete affected-path handling.

Apply masks to explicit projections, filter/sort references, evidence coverage, schema visibility and actual write field paths. Query allow alternatives retain DALgo's conservative **intersection**, now intersecting each include-minus-exclude set; point/write decisions retain the first deciding rule. No union of different policies' includes can defeat another's exclusion. For wildcard reads recurse/project safely before emitting data. A new column matching include:["*"] is allowed unless excluded; explicit legacy field lists continue to deny unlisted new fields. The UI must show this difference instead of promising that every future column defaults denied.

C2 adds restriction kind `field_mask`, representation `mask`, with required mask:{include,exclude?} and no expression/fields/omissionReason. A private or oversized mask becomes a coalesced reference, with kind field_mask or opaque, following the same privacy rules. Existing field_allowlist/fields variant remains. Collection gates are static operation decisions/blockers; no residual mask is needed for the single resolved collection. Return `ACL_COLUMN_DENIED` for excluded columns, not a second column taxonomy. Column names in diagnostics still require safe request/evidence disclosure.

## Implementation and acceptance delta

WP1 adds schema/round-trip/golden and negative fixtures for collectionMask/fieldMask, mask restriction variant and ACL_COLLECTION_DENIED. WP2a extends the existing pattern parser/fieldSet and document AST; WP2b evaluates mandatory collection gates. WP3a/b applies exclusion-aware `allows` everywhere, including enumerable projection fast paths, recursive redaction, query references and affected-path validation. Do not use old enumerable optimization if it would bypass exclusions; conservative fallback stays bounded and enforces before result publication. WP4/5 schema discovery applies gates; WP7 forms preserve include/exclude sets and warn about future-field wildcard inclusion. WP8 tests actual protected queries and updates, not mask serialization alone.

T21 (mandatory): (a) sys_a collection excluded while User_a included; direct lower owner enforces despite upper allow; (b) include * exclude secret_* read omits secret_token, explicit selection/filter/sort reject, UPDATE secret_token denied; (c) include address.* exclude address.internal_* recursively hides/denies internal_code and prevents parent replacement/removal bypass; (d) add a new public field and new secret_ field, verify wildcard behavior after schema reload; (e) multiple matched policies/alternatives cannot union away an exclusion; (f) canonical edit→owner persist→restart→Explain→real query→UPDATE keeps both mask sets; (g) private mask details/count remain hidden, and unsupported nested collection/array operations fail closed. Existing T01–T20/V01–V08 remain required.

C7 review must assess these masks in the same current policy engine and reject unsafe enumerable/subtree shortcuts. No query-text matching, arbitrary SQL effects parser or standalone mask service is introduced.

## Post-review contract clarification

The following closes the parent-evidence ambiguity identified in AS-01. The same parsed mask set exposes distinct checks for a projectable parent, an entirely readable value and a permissible scalar query reference. Wildcard projection may recursively redact a parent. Evidence for an object parent requires authorization for its **complete possible subtree**, not merely the currently present leaves; if any descendant exclusion can apply, reject that parent evidence request even when the excluded child happens to be absent. Request authorized concrete descendant paths instead. Never return a redacted parent as a complete present value, and never infer excluded child absence from the projection. Whole-object filter/order/predicate references are unsupported in the scalar query profile.

New masks reject leading/trailing whitespace rather than silently trim it; preserve legacy fields parser behavior. nameMask is one segment; fieldPattern is a dotted sequence of those masks with existing parent/subtree behavior. Sort and deduplicate include/exclude arrays as semantic sets for canonical output. Preserve original legacy fields canonicalization; `address` and `address.*` remain equivalent parent-prefix inclusions as in current DALgo.

For fieldMask writes, enumerate affected paths from pre-image and final candidate below every touched parent, including removed and unchanged touched descendants. Unknown/opaque composite coverage or unsupported arrays yields ACL_ENFORCEMENT_UNSUPPORTED. Reads recursively redact before publication; disable the enumerable fast path for every exclusion-bearing mask in MVP (a safe optimization can follow later). Remote parent replacement requires complete evidence for every affected subtree and cannot rely on changed-path names alone. Explicit query/evidence scalar references must be complete and authorized, not merely projectable containers.

Disclose the entire field_mask include/exclude expression only with owner policy-read authority for the policy and, if private, owner policy-admin authority. Otherwise return not_authorized opaque reference; **never return only the includes as a weaker admission condition**. Public policy-read may intentionally reveal public mask literals despite separately filtered schema metadata. This is the existing public-policy visibility decision, not anonymous disclosure.

The standard UI deny-column action retains an explicit legacy allow-list; advanced mask editing is a separately labelled mode with a persistent explanation that include:* admits future nonexcluded columns. T21 tests both forms after adding columns. Unresolved/nested collection targets under this MVP profile use ACL_ENFORCEMENT_UNSUPPORTED with operation scope, never ACL_COLLECTION_DENIED; the latter only denotes a resolved name failing a collection gate. Migration docs record root-only collection masks.

T21 executes each applicable case via direct InGitDB, embedded OVDB and remote DataTug, and case (e) combines two mandatory owners plus two within-policy alternatives. Add the included-address/excluded-internal child evidence case with that child both present and absent, array/opaque value rejection, whitespace rejection and legacy parent/subtree differential fixtures. T21's document/schema changes are completed in the final contract artifacts before freeze, not left for divergent downstream agents.
