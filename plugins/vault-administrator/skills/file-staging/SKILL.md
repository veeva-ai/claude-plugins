---
name: file-staging
version: "0.1.0"
description: >
  For a Vault admin or developer who needs files in Vault File Staging (VFS) before a document
  or record load, needs to pull staged files back down, or wants to see and tidy what is
  already staged. Use it to upload a folder tree or a single file (including files too large
  for one POST), download, list, delete, move or rename staged paths, and prove a transfer
  landed.
---

# Vault File Staging (VFS)

Moves files onto and off of Vault File Staging and manages what is there. Every VFS endpoint
lives under `/api/{version}/services/file_staging/`. Transfers run through the login
skill against the Vault the user names; the drafted commands are always available for the user
to run instead.

## Scope

Uploads, downloads, listing, delete, move and rename, resumable uploads for large files, delta
sync of a folder tree, and a proof-of-upload report. Out of bounds, with the owner:

- Building the CSV or JSON load input that references a staged path, and running that load:
  Vault Loader. A multi-object migration with dependency ordering belongs to migration tooling,
  not to an admin staging job.
- Attaching one file to one document or record: usually Direct Upload, not VFS. Step 2 says
  when.
- Anything about signing in: login, in the vault plugin.

Staged files sit outside Vault until a load turns them into documents or records. Once they
are Vault documents or records, questions about them belong to the vault plugin's query skill or to the
vault-developer plugin's api skill, not here.

## Handoffs

Sign in and classify through login, then do the transfer here. When the ask continues
into "now load these", stop at the staged paths and name Vault Loader. For endpoint detail
beyond `references/api-endpoints.md`, use the Vault docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_api_reference`) rather than
guessing a parameter.

Read `references/api-endpoints.md` before drafting any command. It has the endpoint list, the
request and response shapes, and the path and size rules that decide whether a command works
at all. `references/bulk-transfer-and-verification.md` covers the folder-tree walk pattern plus
the delta-sync and proof-of-upload workflows, and it holds the support verdicts for those last
two. For anything in the tens of gigabytes or tens of thousands of files, or coming from another
server, read `references/large-transfers.md` first and size the job before drafting it.

## Step 1: Parse the input

Work out what the user is actually asking for. Match it to one of these jobs:

- **Bulk upload a folder tree** to VFS (a local directory, not a single file)
- **Bulk download a folder tree** from VFS to local disk
- **Upload** a single file or folder item
- **Upload, large file** (the user states or implies a file over 50MB)
- **List / browse** a path
- **Download** a single file
- **Delete** a file or folder
- **Move or rename** a file (folders cannot be renamed or moved; recreate and re-upload)
- **Delta sync**: upload only what's new or changed since a prior transfer
- **Proof of upload**: a verification report after a transfer
- **Reference a staged path** from a document or object-record bulk load
- **Collect an extract** Vault wrote to staging: a Loader document extract with source files,
  renditions, or text, and the files that go with it

Pull out whatever they already stated: local root path and VFS path(s), rough size of the largest
files or of the whole tree, whether this is a repeat transfer and therefore a delta candidate,
whether they want a verification report, and whether this feeds a downstream load. One request
often names several jobs. "Sync this folder to VFS and give me a report when it's done" is bulk
upload plus delta sync plus proof of upload. Resolve every job named. Don't quietly collapse them
into one script.

### What Vault itself puts in staging

Staging is not only inbound. A Vault Loader extract run with Include Source Files, Include
Renditions, or Include Text writes those files to staging and gives back a CSV whose `file`
column is the staging-relative path of each one; renditions get one row per rendition type. An
extract caps how many source files and renditions it will write, so check the current numbers
with the Vault docs MCP before promising a single run covers the whole set. The Export Documents
API writes the same way. So a "get me the source files for these documents" ask is usually: run or locate the
extract, read its CSV, then bulk download from the paths it names, which is the tree-download
job with the CSV as the manifest.

Vault lays those files out predictably, always under the requesting user's folder whatever
their profile:

```
u{userID}/{jobID}/{docID}/{major}_{minor}/{original filename}      source file
u{userID}/{jobID}/{docID}/{major}_{minor}/renditions/{type}.pdf    one per rendition type
u{userID}/{jobID}/{docID}/{major}_{minor}/text/text_file.txt       Include Text
```

The CSV `file` column gives these paths relative to the user folder, so a job's whole output
is one subtree, `u{userID}/{jobID}/`, and one recursive listing or download covers it. Object
attachment field exports do not use staging; they come back through the API as a
`{userID}-{jobID}-{object}.tar.gz`, split into parts over 1 GB.

### Retention

Staging is temporary storage with two clocks. Files Vault wrote (extracts) are deleted every 72
hours by default, so "download the extract" is time-sensitive. Files a user uploaded are kept
for one year, then deleted along with empty directories. A Vault refresh clears staging
entirely, and nothing deleted from staging is recoverable. Say which clock applies whenever a
user asks whether something is still there, and never treat staging as a place to keep files.
Confirm both periods with the Vault docs MCP (`mcp__vault-docs-mcp__search_documentation`,
source `vault_help_documentation`) before quoting a number to a user.

## Step 2: Gather missing information

Ask only for what's missing and needed to draft the script. If a placeholder can stand in, use the
placeholder.

- **Local root path and target VFS path**, for any upload or download job. Check whether the local
  tree's relative structure should be preserved under the VFS target (the normal case) or
  flattened.
- **Admin or non-Admin user.** The two take different path forms (see references/api-endpoints.md
  "Path Semantics"). Ask outright if unstated. Don't assume Admin.
- **Vault DNS and API version**, if they want a filled-in script. If they don't have these handy,
  draft with `$VAULT_DNS` and a `{version}` placeholder and say so plainly rather than inventing
  a value.
- **For delta sync**: confirm they want size and timestamp compared against the current remote
  listing, which is the only signal the API gives back (the reference file's "Delta Sync"
  section), and are not expecting a content-hash diff. The API cannot produce one after the fact.
- **For proof of upload**: confirm the output format (CSV or Markdown table by default) and warn
  before generating that resumable-uploaded files can only be checksum-verified in transit, never
  afterward. See the reference file's "Proof of Upload" section.
- **Whether this upload feeds a bulk load** (Create Multiple Documents, object record attachments,
  renditions, a Vault Loader multi-file load). If it does, every staged path becomes a `file`
  column value downstream, in the same Admin or non-Admin path form that put it there.

Bulk and single-file work split cleanly. Anything batched, and anything driven by a CSV or JSON
input listing many records, needs its source files sitting on VFS first and refers to them by
path. One file going onto one document or object record can skip VFS and ride along on that
operation's own endpoint through Direct Upload, which stops at 4GB. VFS takes a single file up to
500GB through a resumable upload, so "is this file too big" has no answer until the destination
is named. Either way this skill stops at the staged path. Building the load input around it, with
its headers, load order, and external IDs, belongs to Vault Loader.

Sign in through the vault plugin's login skill. The Vault CLI has no file staging command, so
every call here goes to the API through `vault api`, which sends the stored session itself
(with `MSYS_NO_PATHCONV=1` under Git Bash; see `vault-cli.md`). Never read the credential
out of the CLI's store to build a call: the CLI already holds the session, and a second hand-built client is a second place
for a credential to leak and a header to be wrong. Never ask for a session ID or
password here. If the user would rather run the commands themselves, draft them
with `-H "Authorization: Bearer $VAULT_ACCESS_TOKEN"` and `$VAULT_DNS`, the two variables login's
Step 5 has the user set in their own terminal.

The Admin or non-Admin question decides the path form, and the agent cannot infer it from the
conversation. Ask. If the user is unsure, `references/api-endpoints.md` "Path Semantics" has the
one-command probe that settles it.

## Step 3: Say where this is going, then run or draft

State the target DNS and its classification from login's check (its Step 4) in one line.
Uploads, listing, and downloads change nothing in the Vault itself, so they run on any Vault
once the target is stated. Delete and move on a `Production` or `UNKNOWN` Vault are handed to
the user as drafted commands rather than run.

## Step 4: Generate the script or commands

Read the matching section of `references/api-endpoints.md`, and of
`references/bulk-transfer-and-verification.md` for any tree-level job, then draft the script(s)
or curl command(s) for the job(s) identified in Step 1, using the gathered inputs from Step 2. A
few things apply across every job:

- Use the exact path form for the user's Admin/non-Admin status (Step 2). Get this wrong and the
  command 404s, which is far and away the most common way a drafted VFS command fails.
- For any file over 50MB, bulk or single, draft the whole three-call resumable sequence (create
  session -> upload part(s) -> commit) rather than just the first call. A partial draft leaves the
  file half-staged. A folder-tree script branches per file on size; it does not pick one method
  for the whole tree.
- For a bulk upload, subfolders have to exist before the files that go in them. Generate the
  folder-create calls first, parent before child, then the transfers.
- For a bulk download, walk the recursive listing first (`recursive=true`), recreate the local
  subfolder structure from its `folder` entries, then download each `file` entry, chunking any file
  past the `Range` cap into multiple range requests and reassembling locally.
- For a delta sync, generate the listing pull first, build the local/remote comparison, and only
  emit transfer commands for files that fail the size/timestamp match. State the comparison
  logic used in the script's own comments so it's auditable later.
- For a proof-of-upload report, generate the local manifest step, the inline per-file
  checksum-on-upload check (single-shot only), the post-transfer listing comparison, and the
  report-writing step, in that order: see the reference file for exactly which columns are
  provable and which must be marked not verifiable.
- For a delete on a folder, ask whether they want `recursive=true`. Without it, deleting a
  non-empty folder does nothing useful.
- If the target path is the Inbox directory, warn about the Staged-document side effect that
  `references/api-endpoints.md` "Path Semantics" describes, including what rename and delete do
  and don't undo.
- If the request only loosely maps onto the reference material and it's unclear which job this
  really is, say so and ask rather than guessing at an endpoint or a workflow.

## Step 5: Confirm

Report back:

- The script(s)/command(s) generated, in order, and what each does in one line.
- Any placeholder left unfilled (`$VAULT_DNS`, `{version}`, `$AUTH`, real file paths) and
  what the user needs to swap in before running them.
- Per file, or per tree if it's uniform: single-shot or resumable, and the size that decided it.
- The path form used (Admin root path or non-Admin user-relative path).
- **If a nice-to-have branch fired** (delta sync, proof of upload), restate its support verdict
  plainly, in the words `references/bulk-transfer-and-verification.md` uses for it. Never let the
  generated artifact imply a stronger guarantee than the API backs.
- Anything else marked `~` in the reference material that applies to this script. Don't bury
  a caveat that decides whether it works.
- If this upload feeds a bulk load, remind the user that VFS only places the files, then
  the load input that references them is a Vault Loader job, not this skill's.
