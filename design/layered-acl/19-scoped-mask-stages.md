# A3 amendment — scoped alternating include/exclude stages

Status: user-directed semantic correction after the A2/S1 review checkpoint. This amendment is not covered by those reviews and is not an implementation authorization. It supersedes the flat "exclusions always win" proposal for new masks. The reviewed legacy fields behavior, policy hierarchy and restrictive composition across owners remain unchanged. Existing frozen review inputs/manifests remain historical evidence; schemas and executable package fixtures must be revised and the delta reviewed before a new contract freeze.

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

Equivalently, start excluded. For a particular name, process the stages from the beginning: if it matches a stage, set its inclusion state to that stage's action and continue; at the first nonmatching stage, stop and keep the last applicable state. This is bounded name matching, not enumeration of a database or schema. A deeper include cannot override a denial in a different mandatory policy or layer.

| Name | Reason | Included? |
|---|---|---|
| apple | Initial include; does not match the exclusion | yes |
| bacon | Initial include, then contains c; no restoring pattern | no |
| abcd | Initial include, excluded by c, restored by ending d | yes |
| acf | Initial include, excluded by c, restored by a*f | yes |
| bdf | Initial include; exclusion never applies | yes |
| zcd | Outside initial selected set, despite matching later patterns | no |

## Proposed portable shape

Use the same stage form for procedure names, collection names and column patterns:

```yaml
stages:
  - include: ["a*", "b*"]
  - exclude: ["*c*"]
  - include: ["**d", "a*f"]
```

Each stage has exactly one action key with a nonempty pattern list. Stage order is semantic and MUST be preserved through editing/serialization. Consecutive entries of the same type form one OR-list and can be canonically merged before evaluating the next action change; they are not successively intersected. The initial stage is include; "all except sys_*" is include:["*"] followed by exclude:["sys_*"]. An omitted mask retains its existing no-extra-restriction meaning; an empty explicit stage list is invalid rather than unrestricted.

There is no fixed semantic number of stages or patterns. Owners still advertise operational input/depth/work budgets and reject excessive requests explicitly; they must never silently truncate the list or drop later exclusions. Final concrete bounds and schema shape belong to the A3 contract update, not an assumption that unbounded computation is safe.

## Pattern grammar correction

The user's *c* and a*f require interior and multiple wildcard positions, which the reviewed S1 one-edge-star restriction rejected. New mask syntax therefore supports ordinary anchored glob `*` at any position within a name/field segment. Each * matches zero or more characters. Consecutive stars, including **d, mean the same as one star (*d); they do not introduce recursive path traversal. This is a stated interpretation of the example, not a request for a new path language. Collection/procedure names remain canonical single segments; column dots separate nested segments. Wildcards never cross those structural boundaries.

Extend DALgo's parsed mask machinery for the new profile, preserving its legacy fields parser/matcher. Use bounded glob matching without backtracking regex construction. Do not enumerate all schema fields to compile away the stages. Parent-object projection versus complete evidence, affected-descendant write checks and scalar-reference restrictions from S1 continue to apply against the final staged decision. A restored child does not imply that the whole parent is readable.

## Impact on plan and review

WP1: replace flat new-mask schemas/restriction DTOs with staged masks, exact validation and round-trip examples; preserve semantic stage order. Include all rows above, adjacent same-type OR grouping, further alternation, wildcard normalization and budget-exceeded rejection.

WP2/3: extend the reusable DALgo mask representation/evaluator; verify equivalent two-stage behavior for existing flat examples; retain legacy fields compatibility and cross-policy intersection. Record per-stage path/index attribution internally for Explain without requiring English text. Public diagnostics preserve private-policy and row-fact protection.

WP4/5/6: transport/persist the same AST, enforce actual collection/column/procedure gate decisions at their owners, never flatten away nested exceptions or permit a higher owner to grant a lower denial.

WP7: ordered stage editor showing each stage's selected parent set; preserve multiple patterns as one OR-list; warn that moving a stage changes meaning. Do not offer an unordered pair of include/exclude boxes as a faithful editor for this representation.

WP8/T21: actual read/write and disclosure tests for include→exclude→include at each owner/hop, nested-field evidence/redaction, and a deeper include that cannot bypass a different mandatory layer's exclusion. Native effects execution remains unsupported in MVP.

The previous ready-for-sign-off assessment applies to the earlier flat-mask revision only. This new mask contract must receive the same independent delta review/reconciliation before implementation dispatch.
