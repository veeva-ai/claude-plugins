# Functions & Options

## ALLVERSIONS / LATESTVERSION

Applied directly to the FROM target, **no parentheses**:

```sql
SELECT id, major_version_number__v, minor_version_number__v
FROM ALLVERSIONS documents
ORDER BY id, major_version_number__v, minor_version_number__v
```

`FROM ALLVERSIONS(documents)` (with parens) is **wrong**, confirmed against a live Vault.

- `documents` returns the latest version by default (no syntax needed).
- `FROM ALLVERSIONS documents` returns every version of every document.
- `SELECT LATESTVERSION {fields} FROM ALLVERSIONS documents WHERE ...` returns the latest
  version that matches the criteria. `LATESTVERSION` lives in `SELECT` and is only meaningful
  combined with `ALLVERSIONS` in `FROM`.
- `archived_documents` is a separate target for archived docs (excluded from `documents`).

VQL has no `GROUP BY` or `COUNT()`. `SELECT id FROM ALLVERSIONS documents PAGESIZE 0` gives a
total version count via the API response metadata, but not a per-document count. Per-document
counts require filtering to one document at a time or aggregating the raw rows client-side.

**Pattern: one result per document (creator / user-filtered queries)**

When filtering documents by creator, role membership, or user ID, use `FROM ALLVERSIONS
documents` with `SELECT LATESTVERSION` to return one row per document at the latest version
that matches the filter. Plain `FROM documents` returns the latest version and applies the
filter to that version only, so a document whose latest version doesn't match drops out;
`ALLVERSIONS` + `LATESTVERSION` evaluates the filter across every version and returns the
latest one that matches.

```sql
SELECT LATESTVERSION id, name__v, status__v, type__v, document_number__v
FROM ALLVERSIONS documents
WHERE created_by__v = '12345'
AND status__v != STEADYSTATE()
```

`LATESTVERSION` in `SELECT` is only meaningful when combined with `FROM ALLVERSIONS`. Using
it without `ALLVERSIONS` has no effect.

## Relative date literals

Use documented, unquoted VQL date literals, `DATE_LITERAL | DATE_LITERAL:n` syntax:

- `TODAY`
- `LAST_DAYS:n`, `NEXT_DAYS:n`, `DAYS_AGO:n`
- `THIS_MONTH`, `LAST_MONTHS:n`, `NEXT_MONTHS:n`, `MONTHS_AGO:n`
- `THIS_QUARTER`, `LAST_QUARTERS:n`, `NEXT_QUARTERS:n`, `QUARTERS_AGO:n`
- `THIS_YEAR`, `LAST_YEARS:n`, `NEXT_YEARS:n`, `YEARS_AGO:n`

`THIS_MONTH` / `THIS_QUARTER` / `THIS_YEAR` / `TODAY` take **no** `:n` argument. Only the
`LAST_*` / `NEXT_*` / `*_AGO` families take `:n`. `THIS_YEAR:0` is rejected with
`Unknown token [:]`.

Examples:
- "created last month" → `= LAST_MONTHS:1`
- "yesterday only" → `= DAYS_AGO:1`, not `= LAST_DAYS:1`
- "next 5 days excluding today" → `= NEXT_DAYS:5 AND != TODAY`

Don't quote date literals. Only comparison operators are supported with date literals.

## Static dates and datetimes

- Date: `'YYYY-MM-DD'` (quoted; unquoted `2026-02-01` → `Unknown token [-02]`)
- DateTime: `'YYYY-MM-DDTHH:MM:SS.SSSZ'`, milliseconds and trailing `Z` required.
  `'2026-03-31T23:59:59'` is rejected with `date format yyyy-MM-dd'T'HH:mm:ss.SSS'Z' is expected`.

## Booleans

Use lowercase: `true`, `false`.

## Display functions: TONAME() and TOLABEL()

These are **not interchangeable**.

**`TONAME()`:**
- Document fields only. Don't use on object fields.
- `SELECT`: retrieve the field's API name/value.
- `WHERE`: filter by the field's API name, the safest form when working with localized labels.
- Documented for: lifecycle (`lifecycle__v`), lifecycle state (`status__v`), type (`type__v`),
  subtype (`subtype__v`), classification (`classification__v`), document Picklist fields.

```sql
SELECT id, name__v
FROM documents
WHERE TONAME(type__v) = 'sop__c'
```

Don't use `TONAME()` as a way to avoid knowing the API value. The function doesn't authorize
substituting a user-facing label for the API literal.

**`TOLABEL()`:**
- For object fields, not document fields.
- Can be used in `SELECT`, `WHERE`, and `ORDER BY`.
- Documented for: object lifecycle (`lifecycle__v`), object lifecycle state (`state__v`),
  object Picklist fields, object type (`object_type__v`).
- Returns the display label in the user's language.

Use `TOLABEL()` when the request explicitly needs the display label or when filtering on the
documented label form is the right evidenced path.

**AS alias rule:**
- Optional on most fields. Required when the same field appears in one `SELECT` both bare and
  function-wrapped, so the two don't collide on the same response key.
- Alias names: ≤ 40 characters, letters/numbers/underscores only, no two consecutive
  underscores, and not identical to a field name already in the query.
- Prefer simple lowercase aliases (`type_name`, `state_label`).

```sql
SELECT id, type__v, TONAME(type__v) AS type_name
FROM documents
```

```sql
SELECT id, state__v, TOLABEL(state__v) AS state_label
FROM product__v
```

**Other documented functions:**
- `TODISPLAYFORMAT(field)`, a text or number field formatted by its configured format mask
  instead of the raw stored value. `SELECT` only; not supported in `WHERE`.
- `LONGTEXT(field)` / `RICHTEXT(field)`, full Long/Rich Text value (default SELECT truncates
  to 250 chars); SELECT-only.
- `TRIM(field)`, strips the namespace suffix from a picklist value (`hematology__c` →
  `hematology`).

## Document state functions

Use only on `documents` and only with `status__v`. They filter by lifecycle state *type*, not
a specific lifecycle state name, so they work across vaults without knowing the exact
configured state name.

- `STEADYSTATE()`, documents in a steady state.
- `OBSOLETESTATE()`, documents in an obsolete state.
- `SUPERSEDEDSTATE()`, documents in a superseded state.
- `DELETEDSTATE()`, documents in a deleted state (v19.2+).

```sql
SELECT id, status__v
FROM documents
WHERE status__v = STEADYSTATE()
```

Don't use these with object `state__v`. Use `STATETYPE()` for objects instead.

## Object state type filters (STATETYPE)

Use `STATETYPE()` for object lifecycle state-type filtering.

- Use only with object `state__v`, not document `status__v`.
- Argument is a state type name, single-quoted.

```sql
SELECT id, state__v
FROM product__v
WHERE state__v = STATETYPE('initial_state__sys')
```

## Formula and field-type gotchas

- **Formula fields:** Cannot be used in `WHERE`, `ORDER BY`, or `FIND`. Select only; a calc
  error returns `null`. Compute conditions downstream from the raw inputs instead.
- **Long Text / Rich Text:** SELECT truncates to 250 chars, wrap with `LONGTEXT()` /
  `RICHTEXT()` for the full value. In WHERE, effectively only null-checkable or
  CONTAINS/FIND-searchable.
- **Currency:** Returns a string with the currency symbol (e.g. `"$10.00"`).
