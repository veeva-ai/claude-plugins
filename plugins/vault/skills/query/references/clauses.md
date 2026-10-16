# Clauses

WHERE-clause operator mechanics live in `operators.md`; the field/reference/relationship-token
distinction for SELECT/FROM lives in `joins.md`.

## SELECT & FROM

`FROM` names exactly one query target and determines the row grain of the result. `documents`
is a special target name (not `documents__v`) with its own quirks, see `query-targets.md`.

`SELECT` lists local fields, raw reference fields, relationship-token traversals, or
function-wrapped fields (`TONAME(...)`, `TOLABEL(...)`, etc., see `functions-options.md`
for the `AS` alias rule that applies to those).

## SHOW (schema discovery)

`SHOW` is schema-discovery syntax, not a business-answer query:

- `SHOW TARGETS`, lists query targets the authenticated user can query.
- `SHOW FIELDS FROM {target}`, lists queryable fields for that target.
- `SHOW RELATIONSHIPS FROM {target}`, lists queryable relationships, with relationship
  `name`, `target`, and `type` (`inbound` / `outbound`).
- `LIKE` narrows discovery calls: `SHOW TARGETS LIKE '%workflow%'`,
  `SHOW RELATIONSHIPS FROM product__v LIKE '%__cr'`.
- `SHOW FIELDS` / `SHOW RELATIONSHIPS` require `FROM {target}`.
- Don't combine `SHOW` with normal `SELECT`/`WHERE` business-query clauses.
- `SHOW` is something you run (with the user's go-ahead), not a command list you hand back
  to the user to run and report back. Delegating discovery to the user as a command list is
  not the same as gathering evidence yourself.

## WHERE

Structured filtering on exact values, dates, booleans, null checks, reference equality,
picklists. See `operators.md` for the full operator list, several "looks like SQL" operators
behave differently or aren't supported at all in VQL.

## FIND

Use `FIND` only for search intent, keyword/phrase/document-content search.

**Availability:**
- Supported on `documents` and standard-volume Vault objects.
- Not supported on raw Vault objects or in attachment subqueries.
- In v25.3+, supported in subqueries except attachment subqueries.
- Without `SCOPE`, VQL uses `SCOPE PROPERTIES` by default.
- `ORDER BY RANK` is only valid on `documents` with a `FIND` clause.

**Grammar (the most common failure point):**
- Position: `SELECT {fields} FROM {target} FIND ('phrase' [SCOPE ...]) [WHERE ...]`
  `FIND` is a top-level clause after `FROM`, never placed inside `WHERE`
  (`WHERE FIND(...)` → `Unknown token [FIND]`). Any `WHERE` comes *after* `FIND`.
- The search phrase is single-quoted and wrapped in parentheses: `FIND ('cholecap')`.
  Missing parentheses → `Parentheses must be used around FIND search phrase`.
- `SCOPE` goes inside the parentheses. Documents: `PROPERTIES` (default) | `CONTENT` |
  `ALL`. Objects: one or more field names (up to 25, v22.3+). `SCOPE ALL` / `SCOPE CONTENT`
  are documents-only.
- Operators inside the phrase: `AND`, `OR`, `NOT`, `"exact phrase"` (double-quoted within
  the single quotes). Wildcard is `*`; leading wildcards unsupported in v22.3+.

```vql
SELECT id, name__v
FROM documents
FIND ('annual report' SCOPE ALL)
WHERE status__v = 'approved__c'
```

## ORDER BY

`ORDER BY {field} ASC|DESC`, multiple fields comma-separated (priority left→right).

- Cannot sort by a related/reference object field (`product__vr.name__v` in `ORDER BY` →
  `relationship field ... is unexpected in order clause`). Sort on local fields only.
- Formula fields cannot be used in `ORDER BY`.
- `ORDER BY RANK` valid only on `documents` with a `FIND` clause.
- `TOLABEL()` and `FILENAME()` are allowed in `ORDER BY`.

## Pagination: MAXROWS, PAGESIZE, SKIP, PAGEOFFSET

- `MAXROWS {n}`, caps total rows returned, applied after any filters.
- `PAGESIZE {n}`, rows per page.
- `SKIP {n}`, skips the first n results; results start at row n+1.
- `PAGEOFFSET {n}`, paging offset control.
- When `MAXROWS` and `PAGESIZE` are combined, `MAXROWS` must come first.
- Manual paging past 10,000 records errors (v23.1+).

**`PAGESIZE 0` count trick:** returns zero data rows but the API response includes the total
match count, the VQL-native way to answer a count-only question. VQL has no `COUNT()`/
`GROUP BY`, so this is how totals work.

```vql
SELECT id FROM target__v WHERE <condition> PAGESIZE 0
```

## AS

Optional on most fields. Required when the same field appears both bare and function-wrapped
in the same `SELECT` (`TONAME()`, `TOLABEL()`, `LONGTEXT()`, etc.). See `functions-options.md`
§"AS alias rule".
