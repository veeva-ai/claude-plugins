# Sample Query Library

Ready-to-adapt starting points for the most common VQL shapes. Replace anything in `{braces}`
with the real object, field, or value. Every example assumes the field API names given are
correct; if they weren't handed to you directly, confirm them first (SKILL.md Step 2) rather
than guessing at the `__v`/`__c`/`__sys` suffix.

Different from `good-vs-bad-patterns.md` in vqlB, which is specifically about anti-pattern
corrections, these are complete, realistic queries meant as adaptable starting points.

## Canonical relationship shapes

The `team__v` / `players__vr` pair below is an illustrative shape, not a standard Vault object.
It stands in for any parent object and its child relationship token.

**Parent → children, expand (outer join, nested subquery):**
```sql
SELECT id, name__v,
  (SELECT id, name__v FROM players__vr)
FROM team__v
```

**Parent → children, filter (inner join, `IN` subquery):**
```sql
SELECT id, name__v
FROM team__v
WHERE id IN (SELECT team__v FROM players__vr)
```

**Child → parent (dot-notation):**
```sql
SELECT id, name__v, team__vr.name__v, team__vr.city__v
FROM player__v
WHERE position__v = 'Catcher'
```

**Document ↔ Object many-to-many, expand:**
```sql
SELECT id, name__v, type__v,
  (SELECT id, name__v FROM document_product__vr)
FROM documents
```

**Document ↔ Object many-to-many, filter on two related parents at once:**
```sql
SELECT id, name__v, type__v
FROM documents
WHERE id IN (SELECT id FROM document_country__vr WHERE name__v = 'Germany')
  AND id IN (SELECT id FROM document_product__vr  WHERE name__v = 'Cholecap')
```

## Document versions

**All versions of a document type, ordered:**
```sql
SELECT id, major_version_number__v, minor_version_number__v
FROM ALLVERSIONS documents
ORDER BY id, major_version_number__v, minor_version_number__v
```

**Total version count for one document (VQL-native count via `PAGESIZE 0`):**
```sql
SELECT id FROM ALLVERSIONS documents WHERE id = 'YOUR_DOC_ID' PAGESIZE 0
```

**Latest version matching a filter (`LATESTVERSION` + `ALLVERSIONS`):**
```sql
SELECT LATESTVERSION id, name__v
FROM ALLVERSIONS documents
WHERE TONAME(type__v) = 'sop__c'
```

## Document type filtering

```sql
SELECT id, name__v, status__v
FROM documents
WHERE TONAME(type__v) = 'promotional__c'
  AND TONAME(subtype__v) = 'email__c'
```

Document queries filter on labels by default, so a bare `type__v = 'promotional__v'` compares
against the label a user sees, not the API name. Wrap it in `TONAME()` to match the API name,
which is the form that survives localized labels.

## Document usage / view activity (last year)

```sql
SELECT id, document_id__v, document_id__vr.name__v, activity_type__v, activity_date__v
FROM document_usage__v
WHERE activity_date__v = LAST_DAYS:365
```

## Document roles: documents with a user/group in a given role

```sql
SELECT id
FROM documents
WHERE id IN (SELECT document_id FROM doc_roles__sysr WHERE role_name__sys = 'owner__v')
```

Role name literals carry a namespace suffix. Owner is a standard single-user role on every
document, so `owner__v` holds; custom roles are defined per document lifecycle, so confirm
those with `SELECT role_name__sys, user__sys, group__sys FROM doc_role__sys WHERE
document_id = {id}` before filtering on them.

## Matched documents (EDL matching)

```sql
SELECT id, edl_item_id__v, matching_doc_id__v, matching_documents__vr.name__v, edl_item__vr.name__v
FROM matched_documents
```

## Workflow tasks awaiting action

```sql
SELECT label__sys, due_date__sys FROM active_workflow_task__sys
WHERE status__sys = 'available__sys'
```

## Recently modified records

```sql
SELECT id, name__v, modified_date__v, modified_by__v
FROM {object__v}
WHERE modified_date__v = LAST_DAYS:7
ORDER BY modified_date__v DESC
PAGESIZE 1000
```

`LAST_DAYS:n` includes part of today; `DAYS_AGO:n` does not. Pick the one that matches what
was asked: "yesterday" and "the last day" are not the same range.

`modified_date__v` is an object-record field and is rejected on document targets. For documents,
use `version_modified_date__v` for "what changed", `version_creation_date__v` for when a version
was created, or `document_creation_date__v` for when the document itself was created. Pick by
which of the three the question is actually about.

## Orphaned and broken references

**Reference field that should be populated but isn't:**
```sql
SELECT id, name__v
FROM {object__v}
WHERE {reference_field__v} = null AND {scope_condition}
PAGESIZE 1000
```
Use `= null` / `!= null`, not `IS NULL` / `IS NOT NULL`. The SQL forms are rejected.

**Reference populated but pointing at a record that no longer resolves:**
VQL's outbound lookup returns null on the join side when the parent record is gone, archived,
or inaccessible to the querying user, even though the raw field ID is still populated. Any
row where the field has a value but the join field comes back null is a dangling reference.
```sql
SELECT id, {reference_field__v}, {relationship_name__vr}.name__v
FROM {object__v}
WHERE {reference_field__v} != null AND {scope_condition}
PAGESIZE 1000
```
The relationship token follows the `{field}__v` → `{field}__vr` convention; confirm it with
`SHOW RELATIONSHIPS FROM {object__v}` before writing the join.

## Users and group membership

**Users holding the Vault Owner security profile:**

```sql
SELECT name__v, email__sys
FROM user__sys
WHERE (security_profile__sysr.profile_name__sys = 'vault_owner__v')
  AND (security_policy__sysr.authentication_type__sys != 'system_managed__sys')
```

The second condition leaves out users on a system-managed security policy, which suits a
"who are the people" question. It isn't part of filtering by profile, so drop it when you
adapt this sample to another profile unless the user asks to exclude those accounts.

**User-managed groups:**
```sql
SELECT id, name__v, label__v, type__sys
FROM group__sys
WHERE type__sys = 'user_managed__sys'
```

**Members of a named group:**
```sql
SELECT user_id__sys, user__sysr.name__v, user__sysr.email__sys
FROM group_membership__sys
WHERE group__sysr.name__v = 'all_product_experts__c'
```

**Active users with their user-managed group memberships nested:**
```sql
SELECT id,
  (SELECT group__sysr.name__v, group__sysr.label__v
   FROM group_membership__sysr
   WHERE group__sysr.type__sys = 'user_managed__sys')
FROM user__sys
WHERE status__v = 'active__v'
```
The nested subquery result is a separate `data` array embedded per user row, not a flattened
column.

## FAVORITES and RECENT

```sql
SELECT id, name__v FROM FAVORITES documents
```
```sql
SELECT id, name__v FROM RECENT {object__v}
```

These read the user's own Favorites and Recent lists from the Vault UI, so they answer "my
favorite" and "my recent" questions; `RECENT` returns the 20 items viewed most recently. Both
work on documents and object records, and neither works in a subquery or alongside another
target option such as `ALLVERSIONS`.

## Attachments on a document

```sql
SELECT id,
  (SELECT file_name__sys, attachment_id__sys, attachment_version__sys
   FROM attachments__sysr
   WHERE document_version_id__sys = '101_0_2')
FROM documents
WHERE id = 101
```

## Count-only queries (`PAGESIZE 0`)

**Total count matching a condition, no rows returned, only `responseDetails.total`:**
```sql
SELECT id FROM {object__v} WHERE {condition} PAGESIZE 0
```

**Count of matching children per parent row:**
```sql
SELECT id, (SELECT id FROM {child__vr} WHERE {condition} PAGESIZE 0) FROM {object__v}
```
When the question is "how many X", run this form rather than attempting `COUNT()`. VQL has
no aggregate functions in the SELECT clause. The total comes back in `responseDetails.total`,
not as a row.

## Document relationships (annotations/links)

```sql
SELECT source_doc_id__v, relationship_type__v, target_doc_id__v
FROM relationships
WHERE relationship_type__v = 'references__v'
```
