# Endpoint Family Map

The traps worth flagging before making or drafting a command, plus where each family lives.
Every call needs an `Authorization` value (`$AUTH` when running it here, `{SESSION_ID}`
when handing it to a human) and a real `{version}`. The response envelope, pagination shapes,
rate limits, and tracking headers are in the login skill's
`references/auth-and-errors.md` (the vault plugin's login skill); read that for how to interpret any
result before assuming a bad one is this endpoint's fault.

For the request/response shape of any call below, ask the docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_api_reference`) with the family
name in quotes, e.g. "object records batch create" or "job status endpoint".

## Objects / records

Base: `/api/{version}/vobjects/{object_name}` for records, with the metadata under
`/api/{version}/metadata/vobjects`.

- **Create/upsert records in bulk**: up to 500 rows per batch, up to 50MB input. Each row in
  the response `data[]` carries its own `responseStatus`; partial success is normal, check
  every row.
- **User actions and lifecycle state changes on records** are in the Workflows section below,
  which is the single home for both the object and the document side of that.
- **Object record attachments**: the `file` field is a File Staging path, not inline file
  bytes; the file has to be on Vault File Staging (VFS) first (hand that leg to the
  file-staging skill).

A bulk record load goes through Vault Loader (below), not a run of these batch calls. Loader
splits the file into chunks of 500, runs them in parallel, and writes success and failure logs
per task.

## Documents

Base: `/api/{version}/objects/documents`, which is also where renditions, relationships,
binders, checkout, and document workflows hang.

- Bulk-from-file create requires the source file already staged (the `file` column is a VFS
  path); see the file-staging skill.
- A document workflow body names its contents and its participants through parameters that
  vary by workflow. Pull the shape out of the reference before drafting one.

## Workflows

Base: `/api/{version}/objects/objectworkflows`, with the across-many-workflows bulk actions on
`/api/{version}/object/workflow/actions`. This family is about workflows that are already
running: find the instance, see what it accepts, act on it, work its tasks.

- **Find the instance first.** `GET /objects/objectworkflows` needs either `object__v` plus
  `record_id__v`, or `participant` (`participant=me()` for your own). A call with neither has
  nothing to filter on.
- **Action names are per workflow, not a fixed list.** Read `GET
  /objects/objectworkflows/{workflow_id}/actions`, then the same path plus
  `/{workflow_action}` for the prompts that action needs, then `POST` that path to run it.
- **Tasks split single-record from multi-item.** Complete is `/actions/complete` on a
  single-record workflow and `/actions/mdwcomplete` on a multi-item one; reassign is
  `/actions/reassign` against `/actions/mdwreassign`. Picking the wrong one for the workflow's
  shape is the usual first failure.
- **A task action where `esignature` is `true` cannot be initiated through the API at all.**
  Check that flag in Retrieve Workflow Task Action Details before drafting a completion; there
  is no API-side signing path, it has to happen in the UI.
- **A bulk action answers with a `job_id`, not a result.** Poll the job, and expect the
  per-item failures to arrive by email rather than in the response.
- **Object record user actions and state changes.** `GET
  /vobjects/{object_name}/{object_record_id}/actions` lists what that record accepts, and the
  object-level form of the same path with `ids` runs one on up to 500 records. Pull
  `action_name` from the list call rather than composing it. A state change is a user action
  whose `type` is `state_change`, so there is no separate change-state endpoint to hunt for. On
  the multi-record form a required field prompt cannot keep each record's existing value;
  whatever you send is written to every record in the batch, and only the single-record call
  can preserve one.

~ Configuring the workflow itself, its steps, participant controls, and verdicts is MDL and
Admin work, not this skill.

## Direct Data

Base: `/api/{version}/services/directdata`, a high-speed read-only extract of the Vault
published as periodic files rather than served per request.

- **It is not a per-record read.** There is no way to ask it for one record or one object; the
  unit is a published file covering a time window. For a single record or a filtered set, use
  VQL or the object-record endpoints.
- Off by default, and the integration user needs the Direct Data permissions, so an empty list
  can mean not enabled rather than nothing published.
- The extract types, the size at which a file splits, and the part name the download call
  actually wants are in the reference; read it before drafting the download leg.

## Metadata / component MDL retrieval

Two retrieval shapes cover the same underlying components: as MDL, a runnable `RECREATE ...`
statement (`GET /api/mdl/components/{ComponentType}.{name}`), or as JSON/XML (`GET
/api/{version}/configuration/{ComponentType}[.{name}]`).

- The MDL path is versionless, and adding a version segment fails in a misleading way. SKILL.md
  Step 3 has the trap and the exact error.
- Not every component type supports RECREATE generation; check the component type's Supported
  Operations before assuming it does.
- `Object` records are excluded from the JSON collection call; use `GET /metadata/vobjects`
  for those instead.

Authoring, executing, or troubleshooting an MDL script belongs to the vault-administrator
plugin's mdl skill; this section only covers reading back what's already deployed. Within one Vault the
workspace push deploys; between Vaults, a package goes up with Import Package and Deploy Package.

## Jobs

Base: `/api/{version}/services/jobs`, for one job's status and for the jobs still running
across the Vault.

- Status polling is rate-capped per `job_id`, and going faster earns an error rather than a
  faster answer. SKILL.md Step 4 has the interval. Draft a polling loop on a timer, never a
  tight retry.
- Most async operations return a `job_id` immediately and finish out-of-band, so always draft
  the follow-up status poll rather than treating the initiating call's response as proof the
  work finished.

## Vault Loader

Base: `/api/{version}/services/loader`. `POST /services/loader/load` starts a load job from CSV
files already in Vault File Staging, and `POST /services/loader/extract` starts an extract job.
SKILL.md Step 6 has the approval and reporting rules for both.

- **The body is a JSON array of tasks, up to 10 per request.** Each task names `entity_type`
  (`vobjects__v` for records, `documents__v`, `document_versions__v`, and the other document
  entity types for documents), `object` when the
  entity is `vobjects__v`, `action` (`create`, `update`, `upsert`, or `delete`), and `file`, the
  staging path of the CSV (for an Admin, the `/u{user id}/...` form it was staged under). `idparam` (record tasks only) matches an update, upsert, or delete on
  any unique field instead of `id`, and `order` sets the task order.
  A one-task load body:

  ```json
  [{"entity_type": "vobjects__v", "object": "item__c", "action": "upsert",
    "file": "/u12345/loads/items.csv", "idparam": "external_id__v",
    "recordmigrationmode": true, "order": 1}]
  ```

- **Migration flags are per task**: `recordmigrationmode`, `notriggers`, and
  `documentmigrationmode`. Record Migration Mode and Document Migration Mode each need their own
  Vault Owner Actions permission on the loading user. Record Migration Mode lets records load
  into non-initial lifecycle states and lets custom and system fields be set, the record id
  included, and sends no notifications. `notriggers` bypasses all system, standard, and custom
  Java SDK triggers. Document Migration Mode loosens some of the constraints Vault normally
  enforces on a document load and sends no notifications.
- **Rows in one file don't load in order.** Loader splits the file into chunks of 500 and runs
  them in parallel; the logs still map back to the input rows.
- **The response is a `job_id` plus a `task_id` per task.** Poll the job, then read
  `GET /services/loader/{job_id}/tasks/{task_id}/successlog` and `.../failurelog` for every
  task. Both come back as CSV; the `Accept` header only changes the format of an error. The
  success log carries a `rowId` per row, and on an upsert an `event` of `created__sys` or
  `updated__sys`.
- **Read both logs on every task.** The job's status says the job ran, and the failure log is
  where rejected rows show up.
- **An extract body is the same kind of array, up to 10 tasks.** Each task names `entity_type`
  (`vobjects__v`, `documents__v`, `document_versions__v`, `document_relationships__v`, or
  `groups__v`), `object` for records, and `fields`, a required array of field names.
  `vql_criteria__v` filters the rows with a VQL-style expression, and `extract_options`
  (`include_source__v`, `include_renditions__v`, `include_text__v`) adds files for documents.
- **Extract output comes per task.** `GET /services/loader/{job_id}/tasks/{task_id}/results`
  returns the task's CSV, with `/results/renditions` and `/results/text` for the rendition and
  text file paths. A failed task comes back blank rather than with an error, and its failure
  log is only in Admin > Operations > Job Status.
- **Extracted files are temporary.** Renditions, sources, and text land on file staging, and
  staging deletes extracted files every 72 hours by default. Uploaded files are not on that
  timer.

## AI Agent actions

Base: `/api/{version}/services/ai/agent_actions`, which discovers, executes, polls, and
cancels a configured Veeva AI agent action. The Vault AI pages in the reference carry the four
call shapes, the response field names, the status values, and the polling cadence; read them
before drafting a call or a poll loop.

- **Start from the agent list.** `GET /services/ai/agents` returns the standard and custom
  agents, each with the URL of its own `GET /services/ai/agents/{agent_name}` call, which
  returns that agent's class, scope, agent user, model level, and every action it can perform.
  Either call hands you the agent name the action endpoints expect.
- Executing an agent action is a **write**: the action can create or modify records and
  documents. Say which Vault it targets before running it.
- Vault AI is licensed separately, so these endpoints answer at all only in a Vault that has it
  enabled.
- Poll on the documented cadence and back off on `REQUEST_INTERVAL_EXCEEDED` rather than
  tightening the loop.

~ Agent action *outputs* are a common source of the embedded-control-character problem, so parse
status responses non-strictly; see the login skill's auth-and-errors.md. Writing the
Java handlers or the agent MDL behind these actions is Java SDK work, not this skill.

## Logs

Base: `/api/{version}/logs/code` (SDK logs), `/api/{version}/logs/api_usage`, and
`/api/{version}/audittrail` plus the per-record `audittrail` paths. Reading logs that deployed
SDK code already produced is here; writing to the Debug Log from inside that code is
`LogService`, owned by the java-sdk skill.

- **Audit trails**: `GET /audittrail/{type}` returns rows for a type listed by `GET
  /metadata/audittrail`, and a dated window reaches back 30 days at most, while the per-record
  paths have no such limit and default back to the Vault's creation date.
- **API Usage Log**: `GET /logs/api_usage?date=YYYY-MM-DD`, up to 30 days back, always a ZIP.
  It runs about 15 minutes behind, and the docs are explicit that it is not an auditing record:
  a call can be missing from it and still have run.
- **SDK Debug Log**: `GET /logs/code/debug` lists, creates, and deletes sessions, and `GET
  /logs/code/debug/{id}/files` downloads the per-transaction files as a ZIP. **Vault allows
  only one active Debug Log session per user.** A blind `POST` for a user who already has one
  fails with `OPERATION_NOT_ALLOWED "Another resource already exists with
  [log_for_user=<id>]"`. Draft list-first: find the entry whose `user_id` matches the target
  user and `status == "active__sys"`, then either reuse it (if its `class_filters` already
  cover what's needed) or `DELETE` it before `POST`-ing a replacement.
- **SDK Runtime Log**: `GET /logs/code/runtime?date={date}`. Populates roughly 15 minutes after
  an SDK transaction ends, which makes it the wrong choice for iterative debugging; reach for
  the Debug Log first since it's available as soon as the transaction completes.

## Users / groups

Base: `/api/{version}/objects/users` and `/api/{version}/objects/groups`, the legacy user shape
and group membership.

- For `user__sys` records (the modern object-backed shape, v18.1+), prefer the object-record
  endpoint (`GET /vobjects/user__sys/{id}`) over the legacy `GET /objects/users/{id}`.
- `GET /objects/users/me` (Validate Session User) is a quick "am I who I think I am" sanity
  check after authenticating.
- **API Access Tokens** are themselves an object, `api_access_token__sys`; query, retrieve, or
  delete them like any other object record. See the login skill's auth-and-errors.md
  Credential Types section for how they're generated in the first place.

## Picklists

Base: `/api/{version}/objects/picklists`, which lists the picklists, returns one picklist's
values, and edits those values.

- **Retrieve Picklist Values returns active values only.** A load rejected on an unrecognized
  picklist value can still be aimed at a value that exists in the Vault but is inactive, so the
  list you just pulled is not evidence the value is absent.
- Changing a value's name or label reaches every document and object referring to it and can
  break integrations reading values by name. `system` and `systemManaged` picklists refuse
  value edits outright, and the picklists themselves cannot be created or deleted through the
  API at all.
- The write-side calls (add values, rename, reactivate, retitle, inactivate) are in the
  reference.

## Also in the reference

Families this map does not detail. None of them are missing from Vault, so never answer "Vault
has no endpoint for that" about one; read the family in the reference first.

- Binders, Document and Binder Roles, Expected Document Lists, Security Policies, SCIM, Bulk
  Translation, Domain Information: docs MCP.
- Configuration Migration and packaging: Import Package and Deploy Package. The
  vault-administrator plugin's mdl skill owns MDL.
- Sandbox Vaults: docs MCP. The domain endpoints list the Vaults in a domain and the user's domains; see the login skill's Step 2.
- Custom Pages (`/uicode/distributions`): custom-pages.
- Managing Vault Java SDK (`/code/{class_name}`): java-sdk.
