# Operators

These are the operator limitations most likely to produce a subtly wrong query, SQL-familiar
operators that don't behave the same way, or aren't supported at all in certain contexts.

## Comparison & logical operators (WHERE)

Use `WHERE` for structured filtering: exact values, dates, booleans, null checks, reference
equality, picklists.

- Comparison: `=`, `!=`, `<`, `>`, `<=`, `>=`
- Logical/set: `AND`, `OR`, `BETWEEN v1 AND v2` (inclusive range), `CONTAINS (...)`,
  `IN (SELECT ...)`, `LIKE 'prefix%'`

```sql
WHERE document_creation_date__v BETWEEN '2018-11-01' AND '2018-12-01'
```

**Operator limits (Vault rejects otherwise):**

- `OR` cannot connect conditions across different query objects in one `WHERE` (e.g. a local
  field `OR` a related-object field → `[OR] between different query objects is unsupported`).
  Split into separate queries or restructure.
- `NOT` works inside `FIND`, not in `WHERE`. There is no `NOT IN`, see the IN section below
  for the anti-join workaround.
- On raw-object multi-value picklist fields: `>`, `<`, `>=`, `<=` are not allowed; at most
  five `AND` operators; and `AND`/`OR` cannot be mixed on the same multi-value picklist
  field.

## LIKE

Use `LIKE` only for prefix-style text filtering.

```sql
WHERE name__v LIKE 'Annual Report%'
```

`SELECT` statements do **not** support `LIKE '%string'` (leading wildcards), on any target,
and Vault rejects the query. For case-robust or substring search, use `FIND` on `documents` or
a standard-volume object; raw objects don't support `FIND`, so a prefix match is the only
option there.

## CONTAINS

`CONTAINS` is for picklist-style multi-value matching, not generic text search.

```sql
WHERE status__v CONTAINS ('active__v', 'pending__v')
```

Wrong (SQL/SOQL import, not real VQL):
```sql
WHERE CONTAINS(name__v, 'foo')
```

`CONTAINS` is also supported on LongText and RichText fields, but these have otherwise limited
`WHERE` support. Treat them as null-checkable and `CONTAINS`/`FIND`-searchable only.

## Nulls

```sql
WHERE some_field = null
WHERE some_field != null
```

**Do not use SQL `IS NULL` or `IS NOT NULL`.** Vault rejects them.

**Important semantics:** `!=` (and `<`, `>`, `<=`, `>=`) **include rows where the field is
null** in VQL. `WHERE country__v != 'Canada'` returns rows whose country is not Canada *or is
null*. To exclude nulls, add `AND country__v != null`. This is a silent population error. It changes which rows come back without producing any error.

## IN

Use `IN` for nested-query membership, not SQL-style literal lists.

```sql
WHERE id IN (
  SELECT parent_ref_field__v
  FROM child_object__v
  WHERE status__v = 'active__v'
)
```

For a SQL-style literal list (`field IN ('A', 'B')`), use `CONTAINS`; a literal `IN` list is
a parse error:
```sql
WHERE id CONTAINS (3502, 1601)
```

For a multi-value picklist field, `CONTAINS` handles multiple values:
```sql
WHERE TONAME(language__v) CONTAINS ('english__c', 'spanish__c', 'french__c')
```

**SQL-vs-VQL trap:** In SQL the subquery `FROM` is the related table/object; in VQL it must be
the **relationship token on the outer root**, not the related object's API name. See `joins.md`
for the full relationship-token rule.

There is **no `NOT IN`**. An anti-join (exclude rows that have a related match) must be done
by retrieving both sets and differencing them client-side.

## CASEINSENSITIVE and case handling

Field values are case-sensitive by default. `CASEINSENSITIVE()` removes that, but support is
narrow:
- Works only on **standard Vault Object String fields** in `WHERE` (standard data store, not
  raw; custom `__c` String fields are fine).
- NOT supported on `documents`. Use `FIND` for case-robust text on documents.
- NOT supported on raw objects (`user_role__sys` is one) or picklist fields; Vault rejects both
  with an error rather than ignoring the wrapper.
- The filter value still cannot start with `%` (no leading wildcard).

```sql
WHERE CASEINSENSITIVE(product_name__c) LIKE 'wonder%'
```
