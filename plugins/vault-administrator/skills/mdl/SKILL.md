---
name: mdl
version: "0.1.0"
description: >
  For a Vault admin who would rather script a configuration change than click through Admin, or
  who has a config script that failed and wants to know why. Use it for adding a field, reshaping
  an object, fixing a picklist, or repeating the same change across several Vaults, whether or not
  the ask names MDL. Also use it to read back what a component looks like in a Vault today.
---

# Metadata Definition Language (MDL) for Vault Administrators

This skill writes MDL and runs it. A statement that changes a Vault goes out only after the
target's DNS and classification have been stated and the user has said go for that batch.
Statements run one at a time, stop at the first failure, and are read back. Signing in and
creating tokens stay with the user, through the login skill.

## Scope

Reading, authoring, executing, and debugging MDL for configuration components: objects, fields,
picklists, doctypes, lifecycles, workflows, security rules, and the content-bearing types in
Step 5. Out of bounds, with the skill that owns each:

- Java SDK source that an MDL statement references: the vault-developer plugin's java-sdk skill.
- Moving components between Vaults as a package: Admin > Deployment > Outbound and Inbound
  Packages, or Import Package and Deploy Package.
- A question about record or document data rather than configuration: that is VQL, not MDL.
- A one-off load or extract of record data: that is Vault Loader, not MDL.
- A checkbox under Admin > Settings, where no component changes: not a component change at all.
- Any endpoint beyond the `/api/mdl/*` and metadata calls named in this skill. Those come from
  the docs MCP or api, never from a guess; login's Scope has the rule and the reason.

## Handoffs

Name the owning skill and stop at this skill's boundary rather than approximating the other one's
job. The java-sdk handoff holds however the ask is worded, including "just draft it" or a paste of
half-written Java to finish: SDK source written where nothing grounds it reads as authoritative
when it isn't. Hand back the MDL that registers the component against already-deployed code.

Report, Reporttype and Dashboard drafts, including a clone of an existing report under a new name,
come from the reporting skill. This skill executes them.

## Step 1: Classify what's being asked

- **Read**: the user wants to see or understand a deployed component's MDL.
- **Author**: the user wants a new or modified MDL statement (CREATE, RECREATE, ALTER, RENAME,
  DROP) written for them.
- **Execute**: the MDL is already written and the user wants the runnable command, sync or
  async, plus what the response means.
- **Debug**: an MDL script already failed and the user wants to know why.
- **Decide**: the user isn't sure whether MDL, the Admin UI, or a VPK package is the right tool.

Most requests are Author then Execute. Debug needs the exact script text and the exact error
response first: a paraphrase won't match the failure modes in Step 7, and half of them look
identical once paraphrased.

## Step 2: Reading a component's MDL

Look on disk first. If the config skill has pulled this Vault, the component is already
at `components/{ComponentType}/{name}/{name}.mdl` in that workspace (the directory holding
`.vault/state.json`: the directory the user named, the current project, or its `vault-config/`), and the type's
attribute set is in `.vault/metadata/{ComponentType}.json`. An object's types are their own
components, not part of the object's MDL: `allow_types(true)` on an object means look under
`components/Objecttype/{object}.{type}/` for each type before deciding whether a field change
needs the per-type follow-ups in Step 3. Read those, check the workspace is
current (config's Step 3 check), and only go to the API for a component the
workspace does not have. A change starts with a pull, so the files you edit are Vault's
current ones (Step 6). One pull answers every "what does this look like" question for the
session; per-component retrieves do not.

Without a workspace, start in bulk: one call per component type returns every record of that
type as JSON, with names, labels, and for most types the subcomponents too.

```
GET /api/{version}/configuration/{ComponentType}
```

That answers "which ones exist" and usually "what is on it" without a retrieve per component.
`Object` is the exception; its collection comes from `GET /api/{version}/metadata/vobjects`.
Then retrieve a single component as MDL only when the RECREATE statement itself is the thing
you need:

```
GET /api/mdl/components/{ComponentType}.{component_name}
```

Versionless: no `/api/{version}/` segment, unlike almost every other Vault endpoint. It returns a
plain-text, deployable `RECREATE {ComponentType} {component_name} (...)` block. When a component
type's written reference is thin, this is the fastest way to see its real attribute set: retrieve
a working component and edit what came back instead of writing one from scratch.

Two things to flag while reading. A returned `RECREATE` block is a full snapshot, so resubmitting
it unmodified for a targeted change does more than the user asked (Step 4). And for a
content-bearing component (Step 5) this endpoint hands back the MDL shell only, never the binary.
The JSON/XML sibling, `GET /api/{version}/configuration/{ComponentType}.{component_name}`, is
versioned where the MDL form isn't. Confirm which shape the user wants before picking one.

Every component is also a queryable record in `vault_component__v` (`component_name__v`,
`component_type__v`, `checksum__v`, `modified_date__v`). `component_name__v` holds the API name
without the type prefix: `study__v`, never `Object.study__v`. A subcomponent carries its parent
path in dot notation, so an `Atomicsecurity` row reads
`study__v.study_lifecycle__c.closed_state__c.atomic_security__c`, and a `LIKE` filter for one
object's children should include the namespace (`LIKE 'study__v%'`), since `LIKE 'study%'` also
matches `study_country__v` and `study_site__v`. Snapshot the checksums a script is about to
change and re-query them after: an unmoved checksum proves a silent `SUCCESS` no-op.

## Step 3: Author the statement

Read `references/mdl-syntax-reference.md` before writing anything: five statement forms,
subcomponent syntax, attribute data types and the single-quoted string rule, naming and namespace
conventions, the `IF EXISTS`/`IF NOT EXISTS` operators. Ground every statement against it, since
attribute sets are strict and one plausible-looking attribute fails the whole script.

Naming, quoting, and structure rules that apply to everything written in this skill:
- Component/subcomponent names: lower-case snake_case, ending in a double-underscore namespace
  (`__c` for custom; `__v`/`__sys` are standard/Veeva and mostly read-only), 40 characters or
  fewer excluding the namespace. Prefix a shared type with its feature, such as
  `project_category__c` and not `category__c`, since the names of picklists, object types, and
  the like are shared across the whole Vault.
- String and Enum attribute values are always single-quoted (`label('My Label')`,
  `type('Date')`), never double-quoted, in every statement form. Boolean and Number values go
  unquoted (`active(true)`, `max_length(255)`). A Component-reference value is still a quoted
  string; only its *type* differs from a plain string attribute.
- Every component type has a fixed, type-specific attribute set. Don't invent an attribute name by
  analogy from a different type. Resolve unknowns in this order, stopping as soon as you have the
  answer:
  1. `references/component-type-map.md`, covers the most common types and their required attributes.
     For a `Pagelayout`'s `page_markup` XMLString, also read `references/page-layout-markup.md`
     for the XML element schema before writing any markup.
  2. Live Vault retrieve (Step 2's `GET /api/mdl/components/{Type}.{name}`), when a Vault session
     is active. This is faster and more authoritative than any doc search. One call returns the
     full, real attribute set for that type.
  3. `mcp__vault-docs-mcp__search_documentation` calls, if the reference file and the Vault
     both leave the attribute unresolved. Try at most 2 times and if you can't find it, inform the user.
- When authoring a `Field` subcomponent on an `Object` (whether inline in a `RECREATE Object` or
  via `ALTER Object ... ADD Field`), always include `list_column` explicitly. Vault requires it
  on new fields and omits it when echoing an existing component back, so a copied `RECREATE`
  block will fail on a fresh `ADD Field` with a "Required attribute missing" error if `list_column`
  is not stated. Set `list_column(false)` unless the user specifically wants the field to appear
  as a column in list views.
- ~ `external_id__v` can't be created on a custom object, since `__v` is Veeva-owned, so a net-new
  object needs `external_id__c` with `unique(true)` as its reconciliation key. On a standard
  multi-type object, `external_id__v` still has to be enabled per type with
  `ADD Typefield external_id__v(source('custom'), required(false))`.

On an object with object types enabled, a field added by `ALTER Object ... ADD Field` lands on the
base type only. Every non-base type needs
`ALTER Objecttype {object}.{type} (ADD Typefield {field}(source('custom'), required(false)));`
or the field is absent from those records and a later load drops the values with no error.

A lookup field is a two-part construct, and it's the field type that most often stalls for lack of
context, so read before writing. Part one is a reference (Object-type) field that defines the
relationship to the target object: `relationship_type('reference')`, `object('{target}')`, an
outbound name (`relationship_outbound_name('{name}__cr')`), and the inbound name and label that
show on the target. Part two is the lookup field, which reads a value across that relationship:
`lookup_relationship_name` names the relationship to follow (an existing reference field's outbound
name, e.g. `meeting__cr`), and `lookup_source_field` names the field on the target object to pull
(e.g. `related_project__c`). If a reference field already points at the target, reuse its outbound
name as the `lookup_relationship_name` instead of authoring a duplicate relationship.

Two workspace reads (Step 2) make a lookup authorable without asking the user for any of it: the
current object's MDL, to avoid a field-name or order collision and to see its object types; and the
*target* object's MDL, to pick a `lookup_source_field` that actually exists there (and, if you also
have to create the reference, an unused outbound name). The exact attribute set is strict and ~
varies by subtype, so retrieve a real lookup field from the target's app as your template
(Step 2's retrieve-and-edit) rather than composing the block from memory, and confirm any
unfamiliar attribute against `mcp__vault-docs-mcp__search_documentation`. The per-type Typefield
follow-up above applies to a lookup field the same as any other.

Run every statement, one you wrote or one the user pasted, through this shape checklist before
calling it correct:
- Outer parentheses wrap the whole attribute and subcomponent list, and a semicolon closes the
  statement: `ALTER Object x ( ADD Field y(...) );`
- Commas separate every attribute and every subcomponent inside those parentheses, with no
  trailing comma before a closing paren.
- String and Enum values single-quoted, Boolean and Number values bare.
- Component names in the form the type uses: dot-notation for a child of its parent
  (`my_object__c.my_type__c`), and a fully qualified Java class name for a code component such as
  `Recordtrigger`, whose logic lives in `source_code` rather than in attributes
  (`references/component-type-map.md`).
- The parser rules in `references/mdl-syntax-reference.md` hold: no comments, no adjacent string
  literals, no standard-namespace subcomponent declarations, and no empty parentheses unless the
  intent is to clear that attribute.

## Step 4: RECREATE vs ALTER (which one, and why it matters)

RECREATE is an upsert: it creates the component if it isn't there, and otherwise replaces it to
match the statement exactly, so anything live the block doesn't restate gets dropped. It's what
Vault's own generated VPKs use, and it's right for "make this Vault match this definition," but it
also destroys configuration made outside the script, including whatever another admin changed
since. ALTER changes specific attributes or subcomponents (`ADD`, `MODIFY`, `RENAME`, `DROP` on
subcomponents, plus direct attribute changes) and leaves the rest alone, the safer choice for "add
this one field" or "change this one label."

- **DROP** deletes the component outright, and per the docs any subcomponent records under it.
  Irreversible from the API's point of view, recoverable only by building it again.
  ~ Whether Vault holds any grace period or undo for a DROP isn't documented. Treat it as final,
  and confirm with the user before generating one for anything that isn't trivially recreatable.
  Walk reverse dependencies over every component type before dropping, `Reporttype` included:
  a `Report` that a report type still references fails its DROP with `RPT-FDRO-II-33023`.
  A `DROP Field` or `DROP Object` gets the same data count as a delete in a plan (Step 6): Vault
  refuses a field delete with `OBJ-FREC-ONA-7212` while any record holds a value in it, and an
  object drop while the object has records. The inactivate-or-delete question in Step 6 applies here too.
- **RENAME** works on custom (`__c`) components only, fails if the target name is already taken,
  and touches nothing else.
- **`active(false)`** is not a uniform soft delete. On `Object`, `Objecttype`, `Picklist`,
  `Objectlifecycle`, `Objectworkflow`, `Atomicsecurity`, `Job`, `Sharingrule`, `Tab`, and
  `Workflow`, inactivating the parent leaves its subcomponents active, and on no type does
  reactivating a parent reactivate its children. Some types can't be inactivated at all: `active`
  is not editable on `Report`, so a report is dropped or left alone. The type's `.vault/metadata/{ComponentType}.json`
  says which attributes are editable.

A `Pagelayout` holds its entire layout in the single `page_markup` XMLString attribute, so there is
no section-level subcomponent to ALTER: adding a section means retrieving the current markup,
editing it, and resubmitting the whole markup using `RECREATE` or `ALTER`.

Default to ALTER for a targeted "add/change/remove X." Reach for RECREATE only when the user
explicitly wants the component to end up matching a full definition, porting from another Vault or
working from a retrieved snapshot. Say which one you chose and why.

`IF EXISTS` / `IF NOT EXISTS` guard a statement so it no-ops when the target state already holds,
which is genuinely useful for a script meant to run unchanged across several Vaults (syntax in the
reference file). Offer it only when the user describes exactly that. The rest of the time it
swallows the "wait, that component didn't exist" signal you wanted to see.

## Step 5: Components with content (the checksum-bearing flow)

Three component types carry binary content as part of their definition: `Formattedoutput`,
`Overlaytemplate`, `Signaturepage`. The file itself does **not** go through `/api/mdl/execute`;
the MDL statement only references it by name. Two calls:

1. `POST /api/mdl/files`, binary as `multipart/form-data`. The response returns `name__v` (the
   staged file's reference name), `format__v`, `size__v`, and a `sha1_checksum__v`: compare it
   against a locally computed SHA-1 to confirm nothing corrupted in transit before referencing
   it.
2. Reference that `name__v` in the component's `template`/`file`-typed attribute inside a normal
   `CREATE`/`RECREATE`/`ALTER` statement submitted to `/api/mdl/execute`. Once referenced, Vault
   no longer serves the file back from staging; retrieve it afterward via
   `GET /api/mdl/components/{ComponentType}.{name}/files`, which returns the same
   `sha1_checksum__v` for verification.

Don't confuse any of this with the `.md5` file that ships alongside a `.mdl` inside a hand-built
VPK step. That one is a package-integrity check on the MDL script text, generated locally with
`certutil -hashfile` or `md5`, and it belongs to whatever packages the VPK, not to this endpoint.
Full request and response shapes for both content-file calls live in
`references/mdl-syntax-reference.md`.

## Step 6: Execute the script

How the script runs depends on whether the user has the Vault CLI at 26.2.4 or later
(`vault version`). No answer, or anything older than 26.2.4, is a missing CLI: offer to install it
per rule 3 in `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` before running anything, and take
the API path only after a no or a failed install. That file also has the CLI's commands, its
rules, and its known limits. Whichever path, the production check from the vault plugin's login skill comes first,
the target DNS and classification are stated with the batch, the user says go for that batch,
and `--yes` is never part of it. Then run the statements yourself, one at a time. Pass each
one in a way that keeps embedded quotes intact: a bash loop over files with `"$(cat "$f")"`,
or the execute endpoint with `--data-binary` from a file. Windows PowerShell 5.1 strips the
double quotes inside an argument on the way to a native exe, so layout and workflow XML
arrives broken. Stop at the first failure and say what landed, since MDL is not transactional,
then read every component back.

Pick the path by what the change is:

- A change to components, or new ones, with the CLI: the workspace loop below. It is the
  default for new components too, since the files carry the XML that statements cannot.
- A change the user asks to run as a statement, or a quick read-back probe: `vault mdl`, one
  statement at a time.
- No CLI: the API path at the end of this step.

### With the CLI, in a pulled workspace

When the user names the Vault by a nickname ("my QA sandbox", "DEV"), ask for its DNS and pass
it with `--dns`; no CLI command lists the stored Vaults to match a nickname against. Never offer a
different Vault as the default answer.

If this Vault has no workspace yet, make one the way the config skill's Step 2 says: in the
directory the user names, otherwise `vault-config/` under the current project, with `git init`
and a first commit right after the pull. Not under the home directory unless the user asks.

Pull, author the files, preview the push, push, preview again:

```bash
git -C <workspace> add -A && git -C <workspace> commit -qm "before pull"
vault config pull --dns mycompany-qa.veevavault.com --dir <workspace>
git -C <workspace> add -A && git -C <workspace> commit -qm "after pull"   # a new workspace's first commit
# write or edit the component files
vault config push --dns mycompany-qa.veevavault.com --dir <workspace> --dry-run     # preview: what a push would deploy
git -C <workspace> add -A && git -C <workspace> commit -qm "before push"
vault config push --dns mycompany-qa.veevavault.com --dir <workspace>
vault config push --dns mycompany-qa.veevavault.com --dir <workspace> --dry-run     # expect no changes
```

The pull comes first every time. The preview compares the files with the Vault as it is now, so
in a workspace that is behind, a change made in Vault since the last pull shows up as an update
the push would undo. The commit before the pull is not optional: a pull writes over files you
edited, and git is the only way back. If the workspace is not a repository yet, run
`git init` in it first.

After upgrading the CLI, pull again before the first push. A push from a workspace pulled
with an older CLI has proposed deleting components the workspace did not hold.

If a push ends with errors or deploys only some components, stop there. Commit, read back what
landed, and tell the user what landed and what failed before you pull or push again.
If `${CLAUDE_PLUGIN_ROOT}/reference/cli-interim.md` exists, it has the recovery for components
that depend on each other. When a recovery's pull stops before removing new files you have not
pushed, rule 5 in
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` is the only way through: you committed first, so
rerun the pull with `--yes`, restore the files from git, preview, and push.

A new component is a folder `components/<Type>/<name>/` holding `<name>.mdl`, and each
subcomponent is its own folder under that, laid out the way a pulled one is. When the component
has XML (a lifecycle user action's `rule.xml`, a workflow step's `step_detail.xml`, a page
layout's `page_markup.xml`), find the closest working one in the workspace, copy its folder, and
rename the files and the names inside. Do not retype it file by file, and do not hand the piece
that looks hard to the Admin UI. Reading each file and writing a new one is retyping; copy with
the shell, then rename and replace the names:

```bash
cp -r components/Objectworkflow/<example>__c components/Objectworkflow/<new>__c
# rename <example> files to <new>, then replace the old names inside every copied file
grep -rl "<example>" components/Objectworkflow/<new>__c   # expect nothing left
```
 A new field works the same way: copy a pulled field of the
same type and change its name, label and order. Keep every attribute, even the ones that look
like defaults. A Number field without `max_value` fails the push. A user action that starts a
workflow does only that. If the record has to change state when it is submitted, put a
change-state step at the top of the workflow, copied from one that exists. A user action that
changes state and then runs a workflow moves the record and never starts the workflow.

When the new object is modeled on an existing one ("like the PTO request"), the object is only
half the job. Grep the workspace for the example's name outside its own folders: its subtab
under a `Tab`, the related section on its parent's `Pagelayout`, its `Objectvalidation`s. Copy
the subtab and the related section by default and list the validations in the plan as a
question. Without a subtab and a related section, the records exist but nobody can get to them.
`references/component-type-map.md` has the shape of a user action.

The plan stop comes after the files are written and `push --dry-run` has run, so the go is given
against the preview. When the request itself asked for the deploy ("build and deploy", "push it
when it's ready") and login classified the target as non-production, that request is the go.
Run the preview anyway and stop only if it shows a delete or anything this change did not
touch: a stray edit, or a change made on the Vault since the pull, which the preview lists as an
update the push would undo. Otherwise push, and report what the preview listed with the result.
Any other go that came before the preview covers writing the files; stop again with the preview
before the push.

At the plan stop, before asking for the go, say what the push will create, what it will delete
(the push lists deletes and stops for confirmation, which is the user's decision),
whether the workspace was pulled this session, and which new components depend on each other and
in what order they go. When new components in the push name each other (an object and its
lifecycle, a user action and the workflow it runs), name that trap at the plan stop; if
`${CLAUDE_PLUGIN_ROOT}/reference/cli-interim.md` exists, its dependency row says what to do.
Don't say the push will sort out the order. Point at the changed file paths and the
preview output; do not print whole MDL back through a shell `cat`. A push deploys every changed
file in the workspace, so a stray edit elsewhere goes out with it, which is what the first preview
is for.

When the plan deletes a field or an object, count the data it holds first and give the number at
the plan stop: for a field, the records with a value in it; for an object, its records. One query
answers each; the number is `responseDetails.total` in the query response, and the query skill owns
how to run it and read that total without paging rows:

```sql
SELECT id FROM <object> WHERE <field> != null PAGESIZE 0
SELECT id FROM <object> PAGESIZE 0
```

Expect Vault to refuse the delete while any record holds a value in the field. The push reports it
on that one component as `OBJ-FREC-ONA-7212`, "Cannot delete field [x] from object [y]: Records
exist which have data in this field", and the rest of the push still goes out. Quote that text so the user
recognises it. An object drop is refused the same way while the object has records. Clearing or
moving the data is a record write and the user's to do, not this skill's; the delete waits until the
count is zero.

With data or references in play, say at the plan stop that inactivation is the other path:
`active(false)` on the field or object keeps the data and its history and can be reversed, while a
delete cannot be undone. Deleting is the usual move in a development cycle; inactivating is the
usual way to retire a field or object that has been used. Ask which the user wants and do not pick.

After the user's go for the batch, the skill runs the push itself. A push that carries a delete is the exception: the CLI lists the deletes and asks for
confirmation, a shell without a terminal cannot answer it, and `--yes` is never the answer, so
hand that command to the user to run in their own terminal and read the result back. Either way,
read the push output's errors block, not its last line.

### Ordering a set of new components

Work out the order before handing anything over, from the config skill's scoped set and its
dependency graph: picklists and other leaves, then objects, then what hangs off the objects
(lifecycles, validations, workflows, document fields, page layouts, tabs).

An object and its lifecycle name each other, so author the lifecycle in the same change as the
object. Creating an object whose `available_lifecycles(...)` names a lifecycle the Vault does
not have yet still succeeds: Vault creates a placeholder
lifecycle of that name, with Active and Inactive states and their state-type associations, and
says so in its description ("a placeholder lifecycle created by vault as part of configuration
migration"). It is meant to be replaced. So a component being present on the target proves
nothing about its content. With the CLI, the workspace carries the real lifecycle: pull, edit
the lifecycle files, and push, as in the workspace loop. Without the CLI, or to recover from a
placeholder that already exists, read the lifecycle back and keep a copy of what is there. Then
send the real lifecycle as a `RECREATE Objectlifecycle` with its own states, and follow it with a `RECREATE Lifecyclestatetypeassociation` for each association the
placeholder made, pointing `object_lifecycle_state` at the new initial and complete states. Keep
the names Vault gave those associations. It derives them from the lifecycle name, so read them
from `vault_component__v` with a `component_type__v = 'Lifecyclestatetypeassociation'` filter
rather than working them out.

For a component being copied from another Vault, pull that Vault into its own directory and copy
the component's whole folder across, sidecars included. An approval flow for a new object starts
the same way: copy the folder of a workflow that already does that job and rename its object,
lifecycle, states, steps, and labels, rather than writing the step XML from nothing.

MDL is not transactional. When a run stops partway, say exactly which components are now on the
Vault and which are not, check the content of the ones that are, and carry on from there rather
than starting over.

### With the CLI, one statement at a time

```bash
vault mdl "ALTER Picklist color__c ( ADD Picklistentry teal__c ( value('Teal'), order(4), active(true) ) );" --dns mycompany-qa.veevavault.com
```

One statement per call, as a single argument. MDL strings use single quotes, so wrap the
statement in double quotes and escape any double quote inside it; a multi-line statement goes on
one line. The command prints "MDL executed." and nothing else: no `components_affected`, no
warnings. That line means Vault accepted the call, not that anything changed, so every `vault mdl`
is followed by a read-back of the component (Step 2, or the checksum query there). A script of
several statements is several calls in dependency order, which also makes the non-transactional
caveat below visible: each one either landed or did not.

The CLI has no async form. A script that meets the async thresholds below takes the API path.

### Without the CLI, or when the response detail matters

Two endpoints, both taking the same raw MDL text as the request body (must start with `CREATE`,
`RECREATE`, `RENAME`, `ALTER`, or `DROP`):

- **`POST /api/mdl/execute`**: synchronous, and the default. `script_execution` reports
  `warnings`/`failures`/`exceptions`/`components_affected`; `statement_execution` reports
  per-statement `SUCCESS`/`FAILURE` with a message. Read `responseStatus` and
  `script_execution.failures`/`.exceptions`, not just an HTTP 200. A script happily returns 200
  with individual statement failures buried inside it.
- **`POST /api/mdl/execute_async`**: asynchronous, returns a `job_id` immediately; poll
  `GET /api/mdl/execute_async/{job_id}/results` for the outcome. **Required**, not optional, when
  the script operates on 10,000+ records of a `raw` object and does any of: enabling lifecycles,
  enabling/disabling object types, adding/removing a field, changing the max length of a
  variable-length field (Text/Long Text/Rich Text), or adding/removing/reshaping an index. The
  sync endpoint doesn't quietly downgrade for these at scale, it's rejected or times out. Only
  one async raw-object deployment can be queued at a time; the object goes `IN_DEPLOYMENT` and
  can't have fields/layouts/indexes edited until it clears (cancelable via
  `POST .../actions/canceldeployment` while still queued).

Runnable form, with `AUTH` and `VAULT_DNS` set by the block under "No command yet" in
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`:

```bash
curl -X POST -H "Authorization: $AUTH" \
  -H "Content-Type: application/json" \
  --data-binary @mdl_script.txt \
  "https://$VAULT_DNS/api/mdl/execute"
```

No `{version}` segment; both endpoints are versionless. MDL statements are also not transactional,
not even inside one script: if statement 3 of 5 fails, statements 1 and 2 have already committed.
Say that out loud on a multi-statement script making related changes, and suggest single-statement
calls when partial application would leave the Vault in a bad state.

## Step 7: Common failure modes

- **"Required attribute [X] is missing" (`GEN-F***-II-1006`-style)**: the attribute set `ADD` and
  `ALTER` accept is often a stricter superset of what a retrieved `RECREATE` block shows for an
  *existing* component. Vault omits defaulted attributes when echoing a component back, and the
  validator still wants them stated on a fresh `ADD`. Check `context` in the error for the missing
  attribute name, and check the type's attribute table for Required markers before adapting
  retrieved MDL rather than assuming the retrieved block was a complete template. `list_column` is
  the usual concrete case: required on an object `Field` and omitted when Vault echoes an existing
  component back, which is how a copied `RECREATE` block fails on a fresh `ADD Field`.
- A missing dependency shows up when a statement references another component, a `Field` naming
  a `Picklist` or an `Objectaction` naming a `Recordaction` class, and that dependency isn't
  deployed yet. Order multi-statement scripts so dependencies land before their dependents; it's
  the same job the `.dep` file does inside a VPK.
- **`GEN-F***-II-1056`**: malformed XMLString attribute. Validation checks XML well-formedness and
  nothing else, never correctness against whatever schema Vault expects inside it. A well-formed
  block of the wrong shape sails past this check and fails later, or quietly does the wrong thing.
- **Checksum/SHA-1 mismatch on a content component**: the uploaded file's `sha1_checksum__v`
  doesn't match what ends up referenced. Re-upload it. Don't patch the reference by hand.
- **Component-support-matrix mismatches**: some component type pages show blank or "No" for an
  operation that empirically works via the execute endpoint, and at least one code component
  type's page (`Recordtrigger`) shows every operation unsupported while the official Execute MDL
  Script API reference gives a worked `RECREATE Recordtrigger` example. ~ Treat a blank/"No"
  entry as a reason to test against a sandbox first, not proof the operation is impossible:
  retrieving a known-good example of that type (Step 2) is the fastest empirical check.
- Some scripts fail silently instead of erroring: components flagged "Alter: may not be
  supported for some multi-value attributes" will accept a script without complaint and then not
  apply the change. When a script returns `SUCCESS` and the component looks unchanged, re-retrieve
  it (Step 2) before believing the deploy worked.

## Step 8: MDL vs Admin UI vs VPK

MDL automates a change that's complex, repetitive, or has to run identically across several
Vaults. It isn't a replacement for the Admin UI on a one-off manual task, and Vault's own
documentation says so. The split:

- **One-off, low-risk, single Vault, no repeatability need**: Admin UI. Faster, no scripting
  overhead.
- **Same change needs to land on more than one Vault, or needs to be scripted/repeatable/
  reviewable as text**: MDL via this skill.
- **A full package of component + code + data steps needs to move sandbox-to-production with
  dependency tracking, or the user says "VPK" / "migration package"**: that is package
  migration, done through Admin > Deployment > Outbound and Inbound Packages or the package API
  calls.

## Step 9: Present the result and confirm

With the CLI, the change is the edited workspace files and the push; give back the changed paths
and the preview output. Without the CLI, give back the MDL in one code block, statements in
execution order, so it copies as a single script for `api/mdl/execute`; the endpoint runs the commands sequentially, and Step 6's
non-transactional caveat is the reason to offer them one at a time when a partial run would hurt.
Add the exact command to run it, per Step 6: the workspace loop or `vault mdl` when the user has
the CLI, the curl (sync or async) when they do not. With it goes the read-back that proves the
change landed, and on the API path a one-line note on which response fields separate success from
partial failure. Mark with `~`
anything not directly grounded in the docs MCP this session: whether a less-common component
type's supported-operations table can be trusted at face value (Step 7), DROP recoverability, and
any behavior tied to a specific Vault release.
