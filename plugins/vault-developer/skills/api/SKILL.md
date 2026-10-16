---
name: api
version: "0.1.0"
description: >
  For a developer building against the Vault API or debugging a call that came back
  wrong. Use it to find the right endpoint for a job, shape the request, run it against a
  signed-in Vault, and read what the endpoint returned: paging, job polling, logs, and the
  error types an endpoint can raise. Also use it when someone wants a CSV or a set of records
  loaded into Vault ("load this file into my Vault", "load records.csv into Vault", "bulk load
  these rows") or records extracted to a file in bulk: it maps the file to the target object,
  states the load plan, and after the user's go runs it as a Vault Loader job. It also fires
  when someone is building an integration and asks whether Vault has an endpoint for something.
---

# Vault API

Endpoint fluency plus execution. The session comes from the login skill in the
vault plugin; this skill knows the endpoint families, the request shapes, the
traps, and runs the call once the target is stated.

Two different people reach for it. An integration builder wants the right endpoint, the request
shape, and what the response means, and will run the call from their own code; nothing here has
to execute for that to be the whole deliverable. An admin or developer in a session wants the
call run now. For that second case the Vault CLI is the hands wherever it has a command, and this
skill runs raw API calls only for the families the CLI does not cover (Step 3).

## Scope

Any Vault API endpoint family: objects and records, documents, VQL as a call, metadata and MDL
retrieval, jobs, logs, users and groups, AI Agent actions, and the Vault Loader jobs that run an
approved load or extract (Step 6). Out of bounds, with the owner:

- Writing the VQL text itself: the vault plugin's query skill. This skill shapes
  the `/query` call; that one shapes the query.
- Authoring or executing an MDL script: the vault-administrator plugin's mdl skill.
- Moving files through Vault File Staging: the vault-administrator plugin's file-staging
  skill.
- Source-side work beyond mapping one CSV onto its target object, such as joining extracts or
  working out the load order of a multi-object migration. The user brings that done, and this
  skill loads the files.
- Java SDK source, Custom Pages client code, or AI Agent handlers: java-sdk and custom-pages
  here.
- Signing in, credentials, and "which Vault is this": login. That includes a failure
  that surfaces on an API call but is really about the session, `INVALID_SESSION_ID` being the
  usual one: read the error here, hand the fix back there.
- Which Vault the CLI points at, and the Vaults in a domain: login owns that, through the
  domain call in its Step 2.

## Handoffs

Name the owning skill and stop at the boundary. If the vault plugin is not installed, say
so, draft the call with `{SESSION_ID}` and `{vault_dns}` placeholders, and hand it to the user
to run; do not ask for credentials in chat.

## Step 1: Sign in and state the target

Sign-in belongs to the vault plugin's login skill. With the Vault CLI, call Vault through
`vault api`: it sends every call under the signed-in session, `--dns <dns>` names the Vault,
and no shell variable ever holds a credential. Do not read the credential out of the
CLI's store and build a curl call around it.

Without the CLI, the call needs two shell variables, `AUTH` and `VAULT_DNS`, set at the top of
every Bash block that calls the API. `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` has the
two-line version for a user without the CLI. It does not print the credential. An API access
token goes out as `Bearer <token>` and a session ID goes out bare.

If the user is not signed in, hand off to login and stop. Do not ask for a credential here.

Before any write (`POST`, `PUT`, `DELETE`, a job start, a workflow or user action), run
login's classification check (its Step 4) and follow its rule: reads run anywhere; writes run
only against a Vault classified as non-production; on `Production` or `UNKNOWN`, show the
exact call and hand it to the user. State the target and the classification in one line
first, even on a sandbox.

## Step 2: Pick the endpoint family

`references/endpoint-family-map.md` has one section per family: the call shape and the traps.
Load only the family in play. The response envelope, pagination shapes, rate limits, and
tracking headers are in the login skill's `references/auth-and-errors.md`
(vault plugin); read it before interpreting any unexpected result.

For endpoint detail beyond those two files (a parameter, a response attribute, a limit), look
it up with the Vault docs MCP (`mcp__vault-docs-mcp__search_documentation`, source
`vault_api_reference`) rather than asserting it. Name the release when the answer depends on
one.

## Step 3: Make the call

First ask whether the Vault CLI already has a command for this. If `vault version` answers
with 26.2.4 or later and the job is one the CLI covers, run the CLI command and stop; do not
rebuild it as a curl. No answer, or older than 26.2.4, on a job the CLI covers means offer to
install it first, per rule 3 in the reference below, and go raw only after a no or a failed
install.
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` lists what the CLI covers today (sign-in,
configuration pull and push, VQL, MDL, packages) and the rules for using it; a command that is
not in that table is treated as missing. `vault api` is for everything the CLI has no command
for: workflows and tasks, picklists, AI Agent actions, audit, binders, security policies, and the
rest of the reference. When the CLI is not installed, the session from Step 1 carries every
family, and the same call drafted with placeholders is what an integration builder takes away.

`vault api [--dns <dns>] [-X <method>] [-H "Key: Value"] [-d <body> | -F "k=v"] [-o <file>] <path>`.
The path starts with `/` and gets `/api/<version>` put in front; `--api-version none` leaves the
prefix off, for the unversioned MDL endpoints. `-d` sends its value as is, so set the
`Content-Type` yourself with `-H`, and `-d @<file>` reads the body from a file. A response that is
not JSON, a CSV log for instance, needs `-o <file>`. Under Git Bash, prefix
`MSYS_NO_PATHCONV=1` (`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` says why).

Standard headers on everything (through `vault api` the CLI sends the session and its own client ID,
so only `Accept` and `Content-Type` are yours to set, and a query goes as `-d "q=<url-encoded query>"`
with the form content type):

- `Authorization: $AUTH`
- `Accept: application/json`
- `X-VaultAPI-ClientID: claude-plugins` (shows up in the API Usage Log)
- `Content-Type` matched to the body: `application/x-www-form-urlencoded` for `/auth` and
  `/query`, `application/json` or `text/csv` for most bulk endpoints, `multipart/form-data`
  for file uploads.

A VQL call (a read, allowed anywhere; the query text itself is the query skill's job):

```bash
curl -s "https://$VAULT_DNS/api/v26.2/query" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Accept: application/json" \
  -H "X-VaultAPI-ClientID: claude-plugins" \
  --data-urlencode "q=SELECT id, name__v FROM widget__c PAGESIZE 10" \
  | python -m json.tool
```

`--data-urlencode`, never `-d`, for anything carrying query text. To count, run the query with
`PAGESIZE 0` and read `responseDetails.total`; VQL has no `COUNT()`.

The same call through the CLI, with the Vault named and the output kept in a file:

```bash
vault api --dns <dns> --api-version none -H "Accept: text/plain" -o out.mdl /api/mdl/components/<Component>.<name>
```

Retrieve a component as MDL with curl, for a session without the CLI (a read):

```bash
curl -s "https://$VAULT_DNS/api/mdl/components/<Component>.<name>" \
  -H "Authorization: $AUTH" -H "Accept: text/plain"
```

That path is versionless. Add a `/vXX.X/` segment and Vault answers `MALFORMED_URL "... does
not exist"`, which reads like a missing component. When a component you know exists reports
that it doesn't, check the path first.

## Step 4: Read the response

- `responseStatus` is `SUCCESS`, `FAILURE`, or `WARNING`. An HTTP 200 with `FAILURE` in the
  body is the normal failure shape; check the body, not the status code.
- `errors[].type` is the thing to match on. `INVALID_SESSION_ID` means the session or token,
  not your call: hand it back to login's Step 6.
- Capture `X-VaultAPI-ExecutionId` from the response headers whenever a call fails and Support
  might get involved.
- Job endpoints return a `job_id`; poll `GET /api/{version}/services/jobs/{job_id}` no more
  than every ten seconds, and read the job's own status, not the poll's `SUCCESS`.
- ~ Endpoints that can return user- or LLM-generated text (AI Agent action outputs, rich-text
  document reads, SDK Debug Log bodies) sometimes embed raw control characters inside JSON
  strings. Strict parsers reject that, `jq` included. Recover with
  `python -c "import json,sys;print(json.dumps(json.loads(sys.stdin.read(), strict=False), indent=2))"`.

## Step 5: Present the result

The call as run, the Vault it ran against, the part of the response that answers the question,
and the next call if there is one. Anything not confirmed against the docs or a live response
gets a `~`.

## Step 6: Run an approved load or extract

A load reaches this skill as a CSV, either load-ready or a raw export that still needs mapping.
The plan for running it is collected here, from the user, before anything is submitted. Submit
it as a Vault Loader job, never as a hand-built batch of record POSTs.
`references/endpoint-family-map.md`, under Vault Loader, has the request body, the logs, and the
traps. Run this whole step in the session itself, never in a spawned agent: the go for each job is
asked here, and the polling and the logs are read here.

When the file is on disk, read it first. A load-ready CSV has field API names as headers, a
reference written as the reference field plus a unique field on the parent
(`product__c.external_id__c`), picklist values and lifecycle states as API names, dates as
`YYYY-MM-DD`, and numbers with no currency sign or thousands separator. When the user's file is a
raw export instead, map it before the plan, with nothing written to the Vault:

- Map each column to a field on the target object from the configuration workspace (the
  vault-administrator plugin's config skill), or from
  `GET /api/{version}/metadata/vobjects/{object_name}` without one. Name any column that has no
  field and will not load.
- Write the mapped rows to a load-ready copy beside the original, and leave the original as it
  was.
- Check every row as it is mapped: required fields filled, unique fields unique within the file,
  and each reference found in the Vault, with the VQL from the query skill. A row that fails is
  left out of the copy and named in the plan by its source key and the reason. An exact
  duplicate row is listed the same way, with one copy kept. Never fill a blank required value or
  a reference the user did not supply.
- A value map that is a judgment call, a source status onto a lifecycle state or a date that
  reads either day-first or month-first, goes in the plan as a proposal for the user to confirm,
  not as a question of its own. A lifecycle state loads into `state__v`; `status__v` is the
  record's active or inactive status.

1. Collect the plan from the user and confirm the signed-in Vault is the target DNS they name in
   it. When this step built the load file, draft the other fields from that work and state them
   for approval. The plan states:
   - the target DNS and its classification, production or sandbox, from login's check
   - the object or document type, the expected row count, and the rows left out, each by key
     with its reason
   - the column mapping and any value maps, when this step built the load file
   - the operation (create, update, upsert, or delete) and its key field, for update, upsert
     or delete
   - the migration-mode flags (record migration mode, no triggers, document migration mode),
     with why each is on or off: a row that lands in a lifecycle state other than the initial
     one, or sets a system field, needs record migration mode
   - the job sequence, when there is more than one job
   - the post-load verification: a VQL count, supplied by the query skill, run after the job

   When a sheet was prepared elsewhere, the user states these fields from it. A plan counts as
   approved only when the user explicitly approves it with every field above given. A request to
   "load this", or handing over files or credentials, is not approval. With a field missing,
   ask for it and stop. If the signed-in Vault is not the named target, hand off to login and
   stop. Neither this skill nor the assistant submits a Loader job or a batch of record writes
   any other way.
2. The CSV has to be in Vault File Staging already, and the task's `file` is its staging path,
   in the same form it was staged under: `/u{user id}/...` for an Admin. If it isn't there, the
   upload is part of the plan: once the plan is approved, hand the upload to the file-staging
   skill and carry on here with the path it staged.
3. Run Step 1's classification check, then restate the approved plan's line for this job: the
   target DNS and its classification, the object, the operation and key field, the row count,
   and any migration flags. Get a go for that job before submitting it, and a new go for each job after. On
   `Production` or `UNKNOWN`, Step 1's rule applies as written.
4. Submit and poll through `vault api`, and poll on Step 4's interval:

   ```bash
   export MSYS_NO_PATHCONV=1
   vault api --dns <dns> -X POST -H "Content-Type: application/json" -d @load.json /services/loader/load
   vault api --dns <dns> /services/jobs/<job_id>
   ```

   `load.json` is an array of tasks, and its shape, migration flags included, is in the Vault
   Loader section of `references/endpoint-family-map.md`. Build it from there, not from a docs
   search.
5. When the job finishes, pull the success and failure logs for every task
   (`vault api --dns <dns> -o failure-<task_id>.csv /services/loader/<job_id>/tasks/<task_id>/failurelog`,
   and `successlog` the same way) and report the failed rows with their errors. Read the failure log whatever status the job finished with, since a
   job that ends `SUCCESS` can still carry failed rows. A task with no failures answers the
   failure-log call with `MALFORMED_URL` and a message that it was successful without any failure
   records; that is an empty log, not an error. The success log proves a row was accepted, not
   what it wrote: its field columns can come back blank for a value that landed, so the values
   are checked by reading the records back.
6. Run the post-load verification from the plan: have the query skill write the VQL count,
   run it against the target, and compare it with the expected row count and the success log.

An approved extract runs the same way as a Loader extract job, minus the staging check. The first
check is an approved extract request with a target the user named. Approved means the same
thing here: the user explicitly approved a request that names the target DNS and its
classification, the object, the fields, and the filter, and a request to "pull this" or a set
of credentials is not approval. Then the same one-line plan
(the object, the fields, and the filter in place of the operation, key field and row count), a go per job, and Step 1's rule on `Production` or `UNKNOWN`, since starting the job counts as a write.
Poll on Step 4's interval, then read each task's output from its results endpoint. Tell the user
that file staging clears extracted files every 72 hours by default, so anything they need from
the extract should come down soon after it finishes.
