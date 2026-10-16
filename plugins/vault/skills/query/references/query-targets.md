# Query Targets

Each query target has its own queryable fields and behaviors. The most common source of errors
is treating targets as if they work like standard SQL tables when they have Vault-specific rules.

## Documents (special target)

`documents` is a special query target, not `documents__v`. This is the most common naming
mistake for new VQL writers.

Key behaviors:
- `documents` returns only the **latest version** of each document by default.
- `FROM ALLVERSIONS documents` returns every version, see `functions-options.md`.
- `archived_documents` holds archived documents, which `documents` leaves out. Document
  archive isn't available in every Vault, and where it's off Vault rejects the target as not
  queryable, so use it only when the user's Vault archives documents.
- Document search often uses `FIND` for content or property search, see `clauses.md`.

**Document-specific field quirks (these differ from object field conventions):**
- `owner__v` is **not queryable** on documents. The owner is a role, reached through
  `doc_roles__sysr` (see Document Roles below). `created_by__v` is the creator, not the owner.
- Creation date is `document_creation_date__v` (not `created_date__v`).
- Last modified is `version_modified_date__v` (not `modified_date__v`).
- `locked__v` (boolean, true when checked out) and `locked_by__v` (reference to `user__sys`)
  are confirmed standard document fields.

**Documents-only operator limits:**
- `CASEINSENSITIVE()` is NOT supported on `documents`. Use `FIND` for case-robust text search.
- Don't use `TOLABEL()` on document fields. Use `TONAME()` instead (see
  `functions-options.md`).

**Many-to-many to objects** goes through `document_{object}__vr` and must use the subquery
shape. Never dot-traverse a many-to-many token from the outer WHERE:
```vql
-- Wrong
SELECT id FROM documents WHERE document_country__vr.name__v = 'Germany'

-- Right
SELECT id FROM documents WHERE id IN (SELECT id FROM document_country__vr WHERE name__v = 'Germany')
```

### Documents example: latest version query
```vql
SELECT id, name__v, status__v, type__v, created_by__v
FROM documents
WHERE status__v != STEADYSTATE()
```

### Documents example: all versions
```vql
SELECT LATESTVERSION id, name__v, status__v, major_version_number__v, minor_version_number__v
FROM ALLVERSIONS documents
WHERE created_by__v = '12345'
```

## Vault Objects

Standard Vault objects (`product__v`, `site__v`, etc.) and custom objects (`product_complaint__c`,
etc.) follow the general VQL rules. Key reminders:

- Object API names end in `__v` (standard) or `__c` (custom).
- Never guess a `__c` object name. Custom names are vault-specific.
- Use `STATETYPE()` for object lifecycle state filtering, not document state functions.
- `TOLABEL()` works on object lifecycle/state/picklist fields (unlike documents where `TONAME()`
  is used instead).

```vql
SELECT id, name__v, state__v
FROM product__v
WHERE state__v != STATETYPE('inactive_state__sys')
```

## Configuration Components (vault_component__v)

Every configuration component in the Vault, whatever its type, is also a row in
`vault_component__v`, so "does this component exist", "what changed since Tuesday", and
"snapshot these checksums before the MDL runs" are ordinary VQL questions, not metadata calls.
The type-specific attributes (an `Atomicsecurity` component's object or lifecycle, a `Field`'s
data type) are not columns here; they live in the component's MDL, which is
`vault-administrator` → `mdl` territory.

- `component_name__v` is the API name without the type prefix: `study__v`, never
  `Object.study__v`. Filter on it, not on `name__v`, when you have an API name.
- A subcomponent carries its parent path in dot notation, so an `Atomicsecurity` row reads
  `study__v.study_lifecycle__c.closed_state__c.atomic_security__c`.
- `component_type__v` takes the component type name exactly as MDL spells it (`Object`,
  `Objectlifecycle`, `Atomicsecurity`, `Picklist`).
- `name__v` is the label, not the API name. `checksum__v` changes when the definition
  changes, and `modified_date__v` is when.
- Include the namespace in a `LIKE` prefix: `LIKE 'study__v%'` is one object's children, where
  `LIKE 'study%'` also matches `study_country__v` and `study_site__v`.

```vql
SELECT component_name__v, checksum__v, modified_date__v
FROM vault_component__v
WHERE component_type__v = 'Atomicsecurity'
  AND component_name__v LIKE 'study__v.%'
```

## Users (user__sys)

Use `user__sys` for user queries within a Vault. Queryable fields include status,
security profile, security policy, and group membership.

```vql
SELECT id, username__sys, email__sys, status__v
FROM user__sys
WHERE status__v = 'active__v'
```

The login field is `username__sys`, and active/inactive is carried by `status__v` with the
picklist value `active__v`, not a boolean.

Use `SHOW FIELDS FROM user__sys` to discover date and membership fields when building
time-based filters (e.g. "who was added this week").

## Domain Users (users)

The `users` target surfaces domain-level user information that is not available via
`user__sys` (`domain_id__v`, `domain_name__v`, `is_domain_admin__v`, `vault_id__v`). It does
not need domain administrator access, but without it the rows are limited to the current
Vault's users. `MAXROWS`, `SKIP`, `FIND`, and functions such as `CASEINSENSITIVE()` and
`TOLABEL()` are not supported on this target; `PAGESIZE`/`PAGEOFFSET`, `ORDER BY`, `LIKE`,
and `CONTAINS` are.

Use `SHOW FIELDS FROM users` to discover what fields are available.

## Groups (group__sys, group_membership__sys)

```vql
-- All groups
SELECT id, name__v, type__sys
FROM group__sys

-- Members of a specific group
SELECT id, user_id__sys, group_id__sys
FROM group_membership__sys
WHERE group_id__sys = '<group-id>'
```

Group type is `type__sys`, not `type__v`. `group_membership__sys` has only three queryable
fields: `id`, `user_id__sys`, `group_id__sys`. Reach the user or group record itself
through the `user__sysr` / `group__sysr` parent relationships.

## Attachments (attachments__sysr)

`attachments__sysr` is a relationship name, not a standalone root target. Access it through a
subquery off the parent object:

```vql
SELECT id, name__v,
  (SELECT file_name__sys, attachment_id__sys FROM attachments__sysr)
FROM product__v
WHERE id = '<record-id>'
```

There is no `id` or `file__sys` on `attachments__sysr`. The file name is `file_name__sys`.
`FILENAME()` is for *Attachment*-type fields on a record, not for this relationship.

## Document Roles (doc_role__sys)

Document role rows live in `doc_role__sys`. The relationship name from `documents` is
`doc_roles__sysr` (note the `s` and `r` at the end).

```vql
-- Documents where the user has a specific role
SELECT id, name__v
FROM documents
WHERE id IN (
  SELECT document_id FROM doc_roles__sysr
  WHERE role_name__sys = 'owner__v' AND user__sys = '12345'
)
```

Role name literals are namespaced. Standard roles (`owner__v`, `reviewer__v`, `approver__v`)
are stable; custom roles are defined per document lifecycle, so confirm those with
`SELECT role_name__sys, user__sys, group__sys FROM doc_role__sys WHERE document_id = {id}`
rather than assuming.

Use `SHOW RELATIONSHIPS FROM documents` to confirm the exact relationship name and
`SHOW FIELDS FROM doc_role__sys` to confirm field names.

## Document Relationships (relationships)

How documents relate to each other lives in the `relationships` target, one row per
relationship, not in a field on `documents`. `relationship_type__v` holds the type:
`basedon__v` for a copy made from another document, `references__v` for links in
annotations, and Vault-specific types beyond those (the Retrieve Document Type Relationships
endpoint lists them). Filter on that literal whenever the question names a kind of relationship.

Each row has a source and a target, and the question decides which side to read. In a Based
On relationship the copy is the source and the original is the target, so "what was copied
from X" filters the target side and "what did X come from" filters the source side. Say which
side you read. `source_doc_id__v` and `target_doc_id__v` hold the IDs, and `source__vr` and
`target__vr` reach each document's fields.

```vql
SELECT source_doc_id__v, source__vr.name__v, target_doc_id__v, target__vr.name__v
FROM relationships
WHERE relationship_type__v = 'basedon__v'
```

A question that names a document by name takes two steps. Find its ID with a query on
`documents`, then filter `source_doc_id__v = {id} OR target_doc_id__v = {id}`, which stays on
one query object. `source__vr.name__v = 'x' OR target__vr.name__v = 'x'` crosses two query
objects, and Vault doesn't support `OR` there.

## Workflows and Workflow Tasks

`active_workflow__sys` holds in-progress workflow instances and `inactive_workflow__sys` holds
completed or cancelled ones. The owner is `owner__sys`, `status__sys` is `active__v`,
`cancelled__v`, or `completed__v`, and `completed_date__sys`, `cancelled_date__sys`, and
`due_date__sys` carry the dates. `owner__sysr` reaches the owner's `user__sys` record. The
workflow's name is `name__sys` and its label `label__sys`; these targets have no `name__v`.

To find the workflow on a given record, start from the item, since `workflow__sys` exists only
on the item targets:

```vql
SELECT workflow__sys, object_name__sys
FROM active_workflow_item__sys
WHERE object_record_id__sys = 'V5X000000001001'
```

Workflow tasks are `active_workflow_task__sys` and `inactive_workflow_task__sys`. The
assignee is `owner__sys`, `completed_by__sys` is who completed it, and `status__sys`,
`assigned_date__sys`, and `due_date__sys` are there too. The documents or records on a
workflow are in `active_workflow_item__sys` and `inactive_workflow_item__sys`. Querying any of
these needs the Workflow Query or Workflow Participate permission.

```vql
SELECT id, label__sys, owner__sys, completed_date__sys
FROM inactive_workflow__sys
WHERE completed_date__sys = LAST_DAYS:7
```

"My tasks" can also mean user tasks, which are records of an object with the User Task class,
and the Activity object tracks both kinds. Neither API name is in the VQL reference, so
confirm the target before writing it (SKILL.md Step 2).

## Jobs (job_history__sys)

`job_history__sys` holds completed job instances, with `job_name__sys`, `job_title__sys`,
`job_type__sys`, `status__sys`, `queue_date__sys`, `start_date__sys`, and `finish_date__sys`.
Scheduled, queued, and running jobs are in `job_instance__sys` instead. A question about jobs
that failed in a window filters `finish_date__sys` for the window and `status__sys` for the
outcome. Vault Help lists the completion statuses by label (Success, Cancelled, Errors Encountered,
Timeout, Failed to Run, Completed due to Inactivity, Missed Schedule), and the field holds them as
upper-case constants: a successful run is `COMPLETED_WITH_SUCCESS`, so "did not succeed" is
`status__sys != 'COMPLETED_WITH_SUCCESS'`. Other statuses follow the same upper-case shape, such as `MISSED_SCHEDULE`; the spelling of any
other status the answer needs comes from a distinct-values probe on the field, not from the label or from
the REST filter values, which are lower-case and differ. Not every Vault exposes this target
to every user; when it comes back as not queryable, the same history is on the Retrieve Job
Histories endpoint (`GET /api/{version}/services/jobs/histories`), which takes a date window and
a status filter with the documented values `success`, `errors_encountered`, `failed_to_run`,
`missed_schedule`, `timeout`, `completed_due_to_inactivity`, and `cancelled`.

## Reports, Dashboards, and Process Views

Reports are records of the Report object, `report__sys`, not documents and not
`vault_component__v` rows. A flash report is a report record in the Flash Report lifecycle
state, so "flash reports" is a state filter on `report__sys`; confirm the state's name before
filtering on it. Each run of a report or dashboard leaves a Report Usage record
(`report_usage__sys`), which is where "recently run" comes from.

`FIND` runs on `report__sys`. Unscoped it matches the report's text fields, the description
included; `FIND ('term' SCOPE name__v)` restricts it to the name. A `LIKE` on `name__v` cannot start
with a wildcard, so a word anywhere in a report's name is a scoped `FIND`, and a prefix `LIKE` is
only for names that start with the word.

Dashboards are records too, but their target name isn't in the VQL reference, so confirm it
first. Process views are configuration: they are `Processview` components, so
`vault_component__v` with `component_type__v = 'Processview'` lists them.

## Binders (binders, binder_node__sys)

`binders` works like `documents`. What a binder contains is in `binder_node__sys`, one row
per node, with `binder__sysr` for the binder, `document__sysr` for the document at a document
node, and `type__sys` for the node type.

```vql
SELECT binder__sysr.name__v, document__sysr.id, document__sysr.name__v
FROM binder_node__sys
WHERE type__sys = 'document' AND binder__sysr.name__v = '{binder name}'
```

## Objects to Confirm Before Naming

Some questions land on standard objects whose API names aren't in the VQL reference. Name the
object by its label, and confirm the target and fields with the user's Vault before writing
them (SKILL.md Step 2).

- Vault-wide counts and usage trends live on the Performance Statistics object. Vault writes
  one record a day with totals (documents, document versions, attachments, archived documents,
  object records, custom objects) and that day's usage counts (logins, document views, report
  views), so read the latest record rather than counting rows elsewhere.
- Documents on legal hold are Legal Hold Item records, children of a Legal Hold record, one per
  document.
- Agent runs, on Vaults with Vault AI, are Agent Instance and Agent Action Execution records.
  The execution records carry input, output, and total token counts, and Vault deletes both
  kinds after 60 days. For trends past that window, Vault Help's "Monitoring Agents & LLM Token
  Usage" page also describes Daily Agent Activity, which Vault keeps.
