# Negative column projection

Negative projection selects a source wildcard and subtracts named columns or
column-name masks. It is a projection operation, not a sensitive-data policy or
access control boundary.

## YAML form

An unqualified projection selects every available column except the listed
names:

```yaml
from:
  name: customers
columns:
  - wildcard:
      exclude:
        - email
        - password_hash
```

With a source alias, `source` qualifies the wildcard. Names inside `exclude`
are implicitly scoped to that source:

```yaml
from:
  name: customers
  alias: c
columns:
  - wildcard:
      source: c
      exclude:
        - email
```

This is the YAML-native equivalent of the compact notation
`c.*-(email)`. DTQL remains YAML-based and does not require a separate textual
parser for this feature.

To exclude every column whose name begins with `billing` or `password`, use
quoted YAML scalars for the masks:

```yaml
from:
  name: customers
columns:
  - wildcard:
      exclude:
        - 'Billing*'
        - 'Password*'
```

This corresponds to the illustrative compact notation
`*-(Billing*, Password*)`; the DTQL representation remains YAML.

## Semantics

- Exclusions are applied to the wildcard's source only.
- A name without `*` matches exactly, preserving case. A mask containing `*`
  matches the entire column name case-insensitively; each `*` matches zero or
  more characters, so masks also work in the middle or at the start of a name.
- Excluded names that do not exist are ignored; the query still succeeds.
- Duplicate exclusions are harmless.
- Remaining columns preserve their normal wildcard order.
- Explicit columns added after the wildcard are not removed by its exclusions.
- The AST retains the requested exclusion names. A later protocol may report
  matched and unmatched exclusions as diagnostic metadata, but unmatched names
  are not warnings or errors.

A defensive query may exclude `password`, `password_hash`, `secret`, and
`api_key` without first discovering which of those fields exist. This reduces
accidental exposure in a result, but it is not a security boundary. Callers must
continue to rely on authorization and sensitive-data policy for enforcement.

The representation is intentionally structural: a wildcard projection is not
expanded into an ordinary explicit column list, and the source plus requested
exclusions remain available to planners and executors.
