# A3 amendment — scoped alternating include/exclude stages

Status: user-directed semantic correction after the A2/S1 review checkpoint. This amendment is not covered by those reviews and is not an implementation authorization. It supersedes the flat "exclusions always win" proposal for new masks. The reviewed legacy fields behavior, policy hierarchy and restrictive composition across owners remain unchanged. Existing frozen review inputs/manifests remain historical evidence; schemas and executable package fixtures are revised below and the delta must be reviewed before a new contract freeze.

## Selected-set semantics

A mask consists of a variable-length list of stages. Each stage holds an include or exclude action and a list of patterns. Patterns within the same stage are ORed. Every following opposite-action stage applies only to the exact set matched by the previous stage, including all ancestor constraints. It cannot select names outside that set. Inclusion may therefore restore part of a preceding exclusion, without becoming a global override.

For the user's example:

```text
include a*, b*
  exclude *c*
    include **d, a*f
```

Let A be names starting with a or b, C names containing c, and D names ending in d or matching a*f. The permitted set is:

```text
(A minus C) union (A intersect C intersect D)
```

Equivalently, first merge maximal adjacent same-action runs and normalize consecutive stars as specified below. The following algorithm applies only to that normalized list; evaluating raw runs is nonconforming. Start excluded. For a particular name, process the normalized stages from the beginning: if it matches a stage, set its inclusion state to that stage's action and continue; at the first nonmatching stage, stop and keep the last applicable state. This is bounded name matching, not enumeration of a database or schema. A deeper include cannot override a denial in a different mandatory policy or layer.

| Name | Reason | Included? |
|---|---|---|
| apple | Initial include; does not match the exclusion | yes |
| bacon | Initial include, then contains c; no restoring pattern | no |
| abcd | Initial include, excluded by c, restored by ending d | yes |
| acf | Initial include, excluded by c, restored by a*f | yes |
| bdf | Initial include; exclusion never applies | yes |
| zcd | Outside initial selected set, despite matching later patterns | no |

## Portable shape

Use the same stage form for procedure names, collection names and column patterns:

```yaml
stages:
  - include: ["a*", "b*"]
  - exclude: ["*c*"]
  - include: ["**d", "a*f"]
```

Each stage has exactly one action key with a nonempty pattern list. Stage order is semantic and MUST be preserved through editing/serialization. Consecutive entries of the same type form one OR-list and can be canonically merged before evaluating the next action change; they are not successively intersected. The initial stage is include; "all except sys_*" is include:["*"] followed by exclude:["sys_*"]. An omitted mask retains its existing no-extra-restriction meaning; an empty explicit stage list is invalid rather than unrestricted.

There is no fixed semantic number of stages or patterns. Owners still advertise operational input/depth/work budgets and reject excessive requests explicitly; they must never silently truncate the list or drop later exclusions. The reference deployment advertises maxMaskStages=1024 and maxMaskPatterns=4096 per policy document before normalization, maxMaskPatternBytes=128; owners may configure other finite supported limits and clients discover them. The schema has variable-length arrays without a semantic maximum. The existing whole-policy/body and execution/dry-run deadlines still apply. Reject overload at validation or return explicit budget failure; never truncate or treat partial mask evaluation as allow. Stages are a flat list, so predicate AST depth16 does not limit stage count.

## Pattern grammar correction

The user's *c* and a*f require interior and multiple wildcard positions, which the reviewed S1 one-edge-star restriction rejected. New mask syntax therefore supports ordinary anchored glob `*` at any position within a name/field segment. Each * matches zero or more characters. Consecutive stars, including **d, mean the same as one star (*d); they do not introduce recursive path traversal. The user confirmed this interpretation; it is not a recursive path language. Collection/procedure names remain canonical single segments; column dots separate nested segments. Wildcards never cross those structural boundaries.

Extend DALgo's parsed mask machinery for the new profile, preserving its legacy fields parser/matcher. Use bounded glob matching without backtracking regex construction. Do not enumerate all schema fields to compile away the stages. Parent-object projection versus complete evidence, affected-descendant write checks and scalar-reference restrictions from S1 continue to apply against the final staged decision. A restored child does not imply that the whole parent is readable.

## Impact on plan and review

WP1: replace flat new-mask schemas/restriction DTOs with staged masks, exact validation and round-trip examples; preserve semantic stage order. Include all rows above, adjacent same-type OR grouping, further alternation, wildcard normalization and budget-exceeded rejection.

WP2/3: extend the reusable DALgo mask representation/evaluator; verify equivalent two-stage behavior for existing flat examples; retain legacy fields compatibility and cross-policy intersection. Record per-stage path/index attribution internally for Explain without requiring English text. Public diagnostics preserve private-policy and row-fact protection.

WP4/5/6: transport/persist the same AST, enforce actual collection/column/procedure gate decisions at their owners, never flatten away nested exceptions or permit a higher owner to grant a lower denial.

WP7: ordered stage editor showing each stage's selected parent set; preserve multiple patterns as one OR-list; warn that moving a stage changes meaning. Do not offer an unordered pair of include/exclude boxes as a faithful editor for this representation.

WP8/T21: actual read/write and disclosure tests for include→exclude→include at each owner/hop, nested-field evidence/redaction, and a deeper include that cannot bypass a different mandatory layer's exclusion. Native effects execution remains unsupported in MVP.

The previous ready-for-sign-off assessment applies to the earlier flat-mask revision only. This new mask contract must receive the same independent delta review/reconciliation before implementation dispatch.

## Exact integration and canonical form

collectionMask and fieldMask both use `{stages:[stage,...]}`. Stored-procedure execution entries use `{class:stored_procedure,namespace,mask:{stages:[...]}}`; DTQL/native class-only entries stay unchanged. Reject duplicate execution entries for the same class/namespace rather than allow a second entry to bypass an exception chain. The A2 flat include/exclude form was never implemented or released; it is not accepted as an ambiguous alternative in the new schema. Its examples migrate exactly to include then exclude stages.

Stage object: exactly one key include or exclude with a nonempty string array; unknown keys reject. First raw stage must include. Before evaluation, merge each maximal consecutive run of the same action by unioning its patterns; this establishes the selected set for the next opposite-action stage. Only then apply scoped stages. Canonical serialization emits the merged run form: it collapses consecutive stars within segments, sorts/deduplicates patterns within each merged stage, preserves order of action changes, and never sorts stages. Persisted canonical policy bytes use that same merged form. Star normalization occurs before field parsing, including the terminal .* handling, not merely when serializing: address.** must decide identically to address.* for the parent address and its descendants. Both JSON and YAML refer to the same normalized AST. Raw formatting/comments remain out of scope.

Scalar name matching uses canonical Unicode characters, exact case and anchored glob semantics. * may occur anywhere and matches zero or more characters in that segment, never a dot/path separator. New masks reject surrounding whitespace, controls, escapes, ? and character classes. nameMask is one canonical procedure/collection segment; fieldPattern is dotted segments with existing parent-prefix matching. A pattern matching a parent continues to select its descendants, but child stage selection and restoration are always evaluated on the concrete field path under all prior selections. Legacy fields parsing/grammar remains unchanged.

C2 field_mask restriction still uses representation mask, but mask now contains stages. Do not emit a flattened include-minus-exclude approximation. Ordinary blockers remain ACL_COLLECTION_DENIED/ACL_COLUMN_DENIED/ACL_CALLABLE_DENIED with owner/policy provenance; the evaluator may retain stage indexes internally, but no required new English text or public diagnostic fields are needed. If a disclosed mask is too large or private, return the existing opaque reference; never disclose only its positive stages.

Complete parent-object evidence remains conservative: if any descendant exclusion can apply under the staged mask, reject whole-parent evidence unless complete-subtree authorization can be proven. MVP need not implement a glob-language inclusion solver; it may conservatively reject such parent evidence even if a later stage would restore every excluded descendant. Concrete authorized leaf evidence remains possible. A parent's being projectable does not make it complete evidence. Scalar-only query references and final affected-descendant write checks remain mandatory. No stage can restore a value denied by another mandatory policy or owner.

The standard deny-column UI continues to offer a fixed explicit fields list. Advanced staged masks show the initial selection and successive scoped exceptions, preserve action-change order and distinguish future-field wildcard inclusion. Stage reordering changes policy meaning; saves validate and persist at the owner with normal revision CAS. Unknown/stale clients must not edit a staged mask through flat controls.

### Composite projection and restored leaves

Masks assess concrete leaf paths and explicitly requested paths. A wildcard projection may emit structural ancestor objects solely to carry authorized restored leaves; these containers do not represent full-object permission. If no authorized descendant remains, omit a denied container entirely. With include *, exclude address, include address.city, wildcard read may return address:{city:...}, and UPDATE address.city may pass the mask even though explicitly reading/replacing/removing address as a whole remains denied. Ordinary action/row/other-owner checks still apply. Whole-container evidence requires complete-subtree authorization. An explicitly targeted parent mutation checks its requested path and all affected descendants, including removed or unchanged touched children. Structural ancestors of a leaf update are not added as an independent veto. Leaf permission is never permission to overwrite an excluded sibling through parent replacement.

The new compiled mask profile must use an arbitrary-position-glob matcher; it must not silently dispatch through the legacy edge-only segmentMatches. Keep the old matcher for legacy fields. Reuse common parsed paths/field-set intersection and enforcement call sites; no separate ACL engine.
