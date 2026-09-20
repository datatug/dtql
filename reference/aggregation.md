# DTQL aggregation reference

DTQL aggregation is YAML-native. Top-level keys follow logical pipeline order,
with `columns` (the projection/SELECT stage) last:

```yaml
from:
  name: orders
where:
  op: ==
  left: {field: status}
  right: {value: paid}
groupBy:
  - field: country
having:
  op: '>'
  left: {field: revenue}
  right: {value: 10000}
orderBy:
  - field: revenue
    desc: true
columns:
  - field: country
  - aggregate:
      function: count
      args: [{star: true}]
    as: orders
  - aggregate:
      function: count
      distinct: true
      args: [{field: customer_id}]
    as: customers
  - aggregate:
      function: sum
      args: [{field: total}]
    as: revenue
```

Projection may be omitted for a grouped query; this selects both grouping
expressions implicitly:

```yaml
from: {name: customers}
groupBy:
  - {field: country}
  - {field: city}
```

Aggregation without `groupBy` uses one implicit group, including on empty
input:

```yaml
from: {name: orders}
where:
  op: ==
  left: {field: status}
  right: {value: paid}
columns:
  - aggregate: {function: count, args: [{star: true}]}
    as: orders
  - aggregate: {function: avg, distinct: true, args: [{field: total}]}
    as: average_distinct_total
```

## Expressions and validation

`groupBy` contains one or more expressions. Aggregate arguments are expressions;
arithmetic uses `{binary: {op, left, right}}`, allowing forms such as
`SUM(quantity * unit_price)`. Supported functions are `count`, `sum`, `avg`,
`min`, `max`, `first`, and `last`. `distinct: true` is supported by `count`,
`sum`, and `avg`.

Qualified fields use separate `field` and `source` keys, such as
`{field: country, source: c}`. This preserves future JOIN compatibility without
treating `c.country` as a physical column name.

With an explicit `groupBy`, a projected expression must either be a grouping
expression or contain an aggregate. With no `groupBy`, a projected expression
must be aggregated. Aliases can be referenced from `having` and `orderBy`;
renderers rewrite them when a target dialect does not permit aliases there.

When `groupBy` is present and `columns` is omitted, the grouping expressions are
the implicit output projection. Aggregate-only queries without `groupBy` use one
implicit group.

## Logical stages

Execution preserves this order independently of provider syntax:

```text
source -> where -> grouping -> aggregation -> having
       -> requested result ordering -> offset/limit -> projection
```

Provider execution ordering used for streaming groups is internal and never
becomes an implicit result order. Result offset/limit is never pushed into a raw
scan below local grouping.

## Values and result types

- `COUNT(*)` counts every row and returns `int64`.
- `COUNT(expr)` and `COUNT(DISTINCT expr)` ignore null.
- `SUM`, `AVG`, `MIN`, and `MAX` ignore null; empty/all-null input returns null.
- `SUM` and `AVG` use finite `float64` accumulation. Non-numeric dynamic values
  are outside their numeric domain and are ignored.
- Arithmetic normalizes numeric operands to `float64`; non-numeric operands and
  division by zero evaluate to null.
- `MIN` and `MAX` preserve the normalized scalar value.
- `FIRST` and `LAST` include null as a value. They execute only when the provider
  declares a stable input order; aggregate-local `ORDER BY` is reserved for a
  later extension.
- Empty ungrouped input produces one implicit row; explicit grouping over empty
  input produces no rows.

Local grouping normalizes numbers to `float64`, timestamps to UTC RFC 3339, and
uses collision-safe typed composite keys. Strings use binary equality. Native
providers can have unavoidable collation differences; capabilities must not
claim native parity when their semantics are incompatible.

## Planning and resource behavior

DALgo first uses full native aggregation when every required capability is
advertised. Otherwise it pushes safe row filtering, uses provider ordering by
group keys for incremental streaming when available, and falls back to a typed
hash map when it is not. Streaming retains one group's aggregate states. Hash
execution retains one state set per group, never all source records.

The current local safeguards cap groups and distinct values per aggregate at
100,000, cap total aggregate states and total distinct values across a query at
1,000,000 each, and cap retained group keys, distinct keys and JSON-encoded
aggregate state payloads at 64 MiB.
Readers return explicit errors when a limit is exceeded, honor cancellation,
propagate provider errors, and close the upstream cursor.

JOIN, ROLLUP, CUBE, GROUPING SETS, window/statistical aggregates, distributed
aggregation, and disk spilling remain out of scope. Qualified field nodes remain
valid so a later JOIN feature does not require an aggregation AST redesign.
