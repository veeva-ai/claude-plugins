# Joins & Relationships

The inner-vs-outer distinction (filtering vs. expanding related data) and the raw-field-vs-
relationship-token distinction are the two most common sources of "technically valid but wrong
results" queries once more than one target is involved.

## Fields, raw references, and relationship tokens

Don't confuse these three things:

**Local scalar field**, an ordinary field on the root target:
```sql
SELECT id, name__v, created_date__v
FROM submission__v
```

**Raw reference field**, returns the *stored reference value* (an ID), not the related
record's name:
```sql
SELECT id, country__v
FROM submission__v
```

**Relationship traversal token**, use for traversing to related fields:
```sql
SELECT id, country__vr.name__v
FROM submission__v
```

Wrong (dot-traversing the raw reference field instead of the relationship token):
```sql
SELECT country__v.name__v
FROM submission__v
```

Wrong for the same reason, an object API name used where a relationship token is required:
```sql
SELECT id
FROM application__v
WHERE id IN (SELECT related_application__rim FROM activity__rim)
```
`activity__rim` is an object name, not a legal relationship token on `application__v`.

**General rule:** raw field name → use as a field value or equality predicate. Relationship
token → use for traversal or subquery `FROM`. If the relationship token isn't explicitly
evidenced (from `SHOW RELATIONSHIPS` or metadata), don't guess it.

## Relationship patterns

### Child to parent (dot notation)

Use dot traversal when the root row already holds the outbound reference:
```sql
SELECT id, country__vr.name__v
FROM submission__v
```
Use the relationship token, not the object API name.

### Parent to child (nested subquery)

Use nested subqueries when the result must stay rooted on parent rows:
```sql
SELECT id, name__v,
  (SELECT id, name__v FROM child_relationship__vr)
FROM parent_object__v
```

### Parent filtering by child rows (IN subquery)

Use `IN (SELECT ...)` when parent rows should be filtered by matching child rows:
```sql
SELECT id, name__v
FROM parent_object__v
WHERE id IN (
  SELECT parent_ref_field__v
  FROM child_object__v
  WHERE status__v = 'active__v'
)
```

The inner `FROM` must be the **relationship name** (e.g. `players__vr`, `doc_roles__sysr`),
not the related object's API name. The inner `SELECT` returns the primary-side key so Vault
can join them via that relationship.

## Relationship metadata → legal query shape

- `outbound` relationship (child points up to parent) → dot traversal: `relationship_name.field__v`
- `inbound` relationship (parent down to many children) → nested subquery in `SELECT`, or
  `WHERE id IN (SELECT parent_lookup FROM relationship_name ...)`

Always use the exact evidenced `relationshipName`, never guessed. Naming conventions are hints
only: inbound names are often plural (`approved_countries__cr`), outbound names often singular
(`product__cr`), document-object M:M names commonly follow `document_{object}__vr`.
A document field that references an object usually carries the same `document_` prefix: from
`documents`, a field `employee__c` traverses as `document_employee__cr.employee_id__c`, and
`employee__cr` returns "Unknown relationship". The prefix is a convention, not a rule, so take the
field's `relationshipName` from `GET /api/{version}/metadata/objects/documents/properties`
rather than building it from the field name.

## Worked examples

The `team__v` / `player__v` pair below is an illustrative shape, not a standard Vault object.
It stands in for any parent and child.

**Parent → Children (inline subquery):**
```sql
SELECT id, name__v,
  (SELECT id, name__v FROM players__vr)
FROM team__v
```

**Parent filtering by children (IN subquery):**
```sql
SELECT id, name__v
FROM team__v
WHERE id IN (SELECT team__v FROM players__vr)
```

**Child → Parent (dot notation):**
```sql
SELECT id, name__v, team__vr.name__v, team__vr.city__v
FROM player__v
WHERE position__v = 'Catcher'
```

**Document ↔ Object (many-to-many):**

Expand, return related objects nested in document results:
```sql
SELECT id, name__v, type__v,
  (SELECT id, name__v FROM document_product__vr)
FROM documents
```

Filter, return only documents linked to a specific product:
```sql
SELECT id, name__v, type__v
FROM documents
WHERE id IN (SELECT id FROM document_product__vr WHERE name__v = 'Cholecap')
```

Wrong, outer-WHERE dot-traversal across a many-to-many token:
```sql
SELECT id FROM documents WHERE document_country__vr.name__v = 'Germany'
```
Vault returns `Invalid lookup relationship`. The M:M bridge cannot be dot-traversed from the
outer query. Enter it with a subquery instead.

## Join semantics

- Related data in `SELECT` (dot-lookup or nested subquery) is an **outer join**: ALL primary
  rows are returned; missing related data comes back as `null` or empty.
- A relationship in `WHERE` (dot-lookup or `IN` subquery) is an **inner join**: primary rows
  with no matching related row are excluded.

## Hard limits (Vault enforces; exceeding them errors)

- At most **10 unique relationships per query**.
- **One subquery nesting level only.**
- **One relationship per subquery or lookup**, don't chain `a__r.b__r.field__v`.
- **No SQL-style correlated subquery predicates**, VQL correlates relationship subqueries
  implicitly by the relationship name in `FROM`.
- **`ORDER BY` cannot sort on reference-object fields**, sort on root target fields only.

## Many-to-many and bridge objects

A many-to-many relationship always goes through a bridge, never a direct parent-to-parent
path. The bridge cannot be dot-traversed from the outer query. To filter by the OTHER parent,
enter the bridge via a subquery and filter inside it.

**Bridge accessed via a relationship token (e.g. `document_product__vr`):**
```sql
SELECT id, name__v
FROM documents
WHERE id IN (SELECT id FROM document_country__vr WHERE name__v = 'Germany')
```

**Bridge accessed as a separate join object (e.g. `approved_country__c`):**
```sql
SELECT id, name__v
FROM product__v
WHERE id IN (
  SELECT product__v
  FROM approved_country__c
  WHERE country__vr.name__v = 'Canada'
)
```

Not legal, outer-WHERE dot-traversal across a bridge:
```sql
SELECT id FROM documents WHERE document_country__vr.name__v = 'Germany'
SELECT id FROM product__v WHERE approved_countries__cr.country__vr.name__v = 'Canada'
```
Vault returns `Invalid lookup relationship` or `Unknown relationship` for these shapes.
