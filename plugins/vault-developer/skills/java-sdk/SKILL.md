---
name: java-sdk
version: "0.2.0"
description: >
  For developers writing Vault Java SDK code: record triggers, actions, job processors, custom
  APIs, and the SDK services they call. Use when Vault needs custom server-side logic that
  configuration alone cannot express, or to scaffold, review, or debug a class that implements an
  SDK entry point.
---

# Vault Java SDK

Writes and reviews Vault Java SDK code: the entry point class, its annotation, its package, and a
body that survives deploy validation and bulk execution. It puts `.java` files in the
developer's project and hands back whatever commands compile or validate them locally. It never
builds a VPK and never deploys. The one call it makes to a Vault is a read-only run of the class's
own VQL through `vault query` before deploy (Step 6). Every other command is theirs to run.

## Scope

In scope: any class that implements a Vault Java SDK entry point interface, the services it calls
through `ServiceLocator`, the platform restrictions that gate deployment, and the SDK data-type
rules that decide whether the code survives its first real run.

Out of scope, by name:

- **The Custom Page three-file shape** (MDL component, React client bundle, and the
  `PageController` that ties them together) goes to the `custom-pages` skill. The Java SDK
  constraints in this skill still apply to any `PageController` Java you write; `custom-pages` owns
  the page-specific wiring those three files must agree on, not an exemption from these rules.
- **AI Tool Type and Context Type runtime handlers, Agent and Agentaction MDL, and
  `AgentActionResultHandler`** go to the `ai-agents` skill, including the RecordTrigger or
  RecordAction that starts an agent and writes the result back.
- **Deploying it** goes to the vault-administrator plugin's config and mdl skills for a workspace
  push, or to a package built by hand and deployed through the API ("Getting code onto a Vault").
  This skill stops at source on disk.

## Handoffs

- Deployment of anything written here: the vault-administrator plugin's config and mdl skills for a
  workspace push, or the Import Package and Deploy Package calls for a package.
- "Does Record Migration Mode alone suppress my trigger during this load?": the answer is in
  `references/platform-constraints.md`.
- A Custom Page: the `custom-pages` skill in this plugin. An AI Agent artifact: the
  `ai-agents` skill. A plain Vault API call against a vault: ground it against the Vault docs
  MCP.

## Step 1: Pick the entry point

Entry point interfaces define how and when Vault runs custom code. Match the described behavior to
one before writing anything, because the annotation, interface, and context object all follow from
this choice and none of them are interchangeable.

| Behavior described | Entry point | Interface |
|---|---|---|
| Runs automatically when a record is inserted, updated, or deleted | Record trigger | `RecordTrigger` |
| Runs automatically when a user role on a record changes | Record role trigger | `RecordRoleTrigger` |
| Runs automatically on a document version event | Doctype trigger | `DocumentTypeTrigger` |
| A user or API caller invokes it on demand against a record | Record action | `RecordAction` |
| A user or API caller invokes it on demand against a document | Document action | `DocumentAction` |
| Runs on a workflow start or task step | Record workflow action | `RecordWorkflowAction` |
| Scheduled or bulk background processing, chunked into tasks | Job processor | `Job` |
| A custom REST endpoint other systems call | Custom API | `WebApi` |
| Consumes a message from an inbound queue | Message processor | `MessageProcessor` |

Record merge event handlers, email processors, and message delivery event handlers are entry
points too, less commonly asked for. Don't treat the table above as exhaustive or rely on memory
for an interface's methods and attributes: always inspect the actual API. When the developer's
project declares the Vault Java SDK as a dependency (a Maven `pom.xml` or a `lib` folder the project
points at), inspect that `.jar`, the exact version they build against. Otherwise use the docs MCP
(`mcp__vault-docs-mcp__search_documentation`). Never search the disk or the home directory for a
`.jar`; a project that does not declare one does not need one, since Vault compiles on deploy.

Two shapes get confused constantly. A trigger fires on its own, as part of the data operation,
with nobody choosing it. An action is chosen: a menu item, a button, an API call naming it. If the
request says "when the record is saved", that is a trigger; "when the user clicks", that is an
action; "every night" or "process the whole backlog", that is a job.

If the described behavior does not match anything in the table, say so plainly rather than forcing
it into the nearest shape. Inspect the interface before committing to one this skill has not named:
the SDK `.jar` the developer's project declares, otherwise the docs MCP
(`search_documentation`, source `vault_java_sdk_javadocs`).

## Step 2: Gather the inputs

Ask for whatever is missing in a single message, not one at a time. Every entry point needs a
class name and enough of a business-logic summary to write a body instead of a stub. Beyond that,
the annotation decides what else is required: an object API name and event list for a record
trigger, an object and label for a record action, a job metadata API name for a job.

Object API names carry a namespace suffix (`__v` standard, `__c` custom, `__sys` system). If the
developer names the object loosely, ask instead of guessing the suffix. After the first question,
offer the escape hatch: "Or type **skip** to generate with safe defaults for anything missing."
The object API name has no safe default. If it is still missing after skip, stop and ask for it
specifically. Nothing else can proceed without it.

## Step 3: Ground the code against platform constraints

Read `references/platform-constraints.md` before writing anything. It covers the JDK allowlist
(the full class-and-method list ships as `references/jdk-allowlist.csv`, grep it for any JDK
type you are not certain is allowed),
the restrictions Vault checks at deploy time (thread safety, no subclassing, no third-party
libraries, forbidden constructs like reflection, multi-threading, and file or console I/O),
`VaultCollections` in place of `new ArrayList<>()`, the governor limits that bulk-unsafe code
trips first, and the SDK-wide data-type rules for VQL results and JSON output. Those apply to
every entry point type, not just triggers. `LogService` usage and how to read the SDK debug log
are in `references/logging-and-debugging.md`; read it when the class needs diagnostics or the
developer is chasing a runtime failure.

The platform restrictions are checked at deploy validation, the data-type rules at runtime, and
none of them by the compiler. Code that ignores them builds clean on the developer's machine and
fails the moment they try to deploy it.

Five rules are worth carrying rather than looking up every time:

- Custom SDK classes live under `com.veeva.vault.custom`. Sub-packages are fine for organization.
- Build collections with `VaultCollections.newList()` and friends. Direct JDK collection
  constructors are rejected at deploy validation.
- Log through `LogService`, never `System.out`. Use the `{}` message-template overload.
- `RollbackException` is the only exception type SDK code may throw or catch.
- Never call a method on an SDK enum. `java.lang.Enum` has no entries on the JDK allowlist, so
  something like `error.getQueryOperationErrorType().name()` fails deploy validation. A query
  failure is reported with `error.getMessage()` and nothing else. The allowlist is the Java SDK
  "Limits and Restrictions" page in the Vault docs (the docs MCP, source
  `vault_developer_documentation`), so check it there.

## Step 4: Resolve the output location

Look for an existing `src/main/java/com/veeva/vault/custom` in the developer's project, or a
`custom` subpackage already holding classes of the type being written. Sub-packages are fine
(`com.veeva.vault.custom.triggers`, or something feature-scoped like
`com.veeva.vault.custom.rim.submissions.triggers`), but do not invent one the developer did not
ask for. No such directory? Ask where the class goes rather than guessing at their project layout.

A Vault CLI workspace (a directory with `.vault/state.json`) keeps code under `javasdk/` by
package path, `javasdk/com/veeva/vault/custom/...`, with no `src/main/java` in front. The CLI
builds the class name from the path under `javasdk/`, so a Maven layout placed there produces
the wrong name. When the developer works in a workspace, that is where the class goes; when
they have a Maven project, it stays in the Maven layout. A VPK built outside the CLI uses the
Maven form inside its own `javasdk/` folder: `javasdk/src/main/java/com/...`.

Check whether the file already exists before writing; never overwrite silently.

## Step 5: Write the class

The worked example this skill carries in full is the record trigger, because it is the entry point
most requests land on and the one whose failure modes generalize. Read
`references/record-triggers.md` for the canonical template, the `@RecordTriggerInfo` attributes,
the data-availability table (which `get` and `set` calls are legal on which event), and the common
patterns: field default, validation-cancel, bulk related-record creation, DELETE handling, and
multi-event guards.

`references/skeletons.md` has compile-ready files, imports included, for a `PageController` that
runs one query and for an AI tool handler that takes no input. Start from those for either shape.

Every other entry point follows the same three-part shape: an `@...Info` annotation carrying the
configuration Vault needs, a class implementing the interface, and one or more methods taking a
context object that exposes the records, parameters, or message the platform handed in. When
writing an entry point other than a record trigger, confirm the annotation's optional attributes
and the context object's methods by inspecting the interface, the SDK `.jar` the developer's
project declares, otherwise the docs MCP (`search_documentation`, source
`vault_java_sdk_javadocs` for the interface, `vault_developer_documentation` for the entry point's
behavior), rather than inferring them from the trigger example. Never invent an attribute name.

While inspecting, from the JAR or the MCP, also check whether the method or annotation attribute
is deprecated. If it is, use the documented replacement instead. Never generate code that calls a
deprecated SDK method. The ones that turn up most often still compile and still work, which is
why they keep getting suggested; treat this table as settled and skip straight to the replacement:

| Deprecated | Since | Use instead |
|---|---|---|
| `RecordService.batchSaveRecords(List<Record>)` | 22R2.3 | `batchSaveRecords(RecordBatchSaveRequest)`, built with `newRecordBatchSaveRequestBuilder().withRecords(list).build()` |
| `RecordService.batchDeleteRecords(List<Record>)` | 22R2.3 | `batchDeleteRecords(RecordBatchDeleteRequest)`, built with `newRecordBatchDeleteRequestBuilder().withRecords(list).build()` |
| `RecordService.saveRecords(List<Record>)`, `deleteRecords(List<Record>)` | 17R2.2 | the same two request-builder overloads |
| `QueryService.query(String)` | 21R3.4 | `query(QueryExecutionRequest)`, built with `newQueryExecutionRequestBuilder()` |
| `QueryService.escape(String)` | | `escapeValue(String, QueryEscapeOverride...)` |
| `HttpService.newHttpRequest(String)`, `newLocalHttpRequest(...)` | 24R1.1 | `newHttpRequestBuilder().withConnectionName(name)` or `.withLocalConnection(userType)`, then `.build()` |
| `HttpService.send(HttpRequest, ...)` | 24R1.1 | `sendRequest(HttpWebRequest, ...)` with the same second argument |

**Non-negotiable, regardless of what the developer's spec says:** the body must be bulk-safe.
Never call `QueryService`, `RecordService`, `HttpService`, `DocumentService`, or any other service
inside a per-record or per-item loop. That includes a loop over `getDocumentVersions()` in a
document action, over `getRecords()` in a record action, and over `getRecordChanges()` in a
trigger. Build the input list (or a single query condition, or a single request payload) across
all items first, make one service call, then apply results back in a second pass. A service call
inside the loop is not slow, it is fatal: the code passes a one-record test and blows the governor
limit the first time it meets a bulk API call, an import, or a multi-select UI action. HTTP
callouts have a second problem on top: they are not part of the transaction, so a loop that fails
partway leaves the remote system holding the first N messages while Vault rolls back. The pattern
for each service is in the bulk-safety section of `references/platform-constraints.md`.

When a `QueryService` query uses the `CONTAINS` operator, use `TokenService` to supply the value
list rather than concatenating strings into the query. Write the placeholder directly in the query
string (`CONTAINS (${Custom.my_list})`), then resolve it by building a `TokenRequest` with
`withValue("Custom.my_list", asList("val1__c", "val2__c"))`. CONTAINS only supports a token
containing a `List`, not a String or other scalar type. Obtain `TokenService` via `ServiceLocator.locate(TokenService.class)` and pass the
built `TokenRequest` to the execution request builder via `withTokenRequest(tokenRequest)`. Vault
expands the list into the correct `('val1__c', 'val2__c')` syntax and handles character escaping.
Token names must start with `Custom.` and be lowercase alphanumeric with single underscores.

Every VQL string the class runs follows the vault plugin's query skill, so load it before writing
one. The rule that catches handler code most often is the list: literal values are matched with
`CONTAINS`, through the token above when the list is a variable, and `IN` takes only a subquery.
`field IN ('a', 'b')` is not valid VQL, and inside a class it surfaces only when the query runs.
Give every query an `onError` that carries `error.getMessage()` back to the caller, in the JSON a
tool returns or in the `RollbackException` a controller throws, so a failure reads as a sentence
instead of an unexpected error with no detail.

## Step 6: Confirm

Before the class goes anywhere near a deploy, run each VQL string it holds read-only with
`vault query "<VQL>" --dns <dns>`, with real values in place of any token, and fix the ones
that fail. A query is a read and writes nothing, but it needs a signed-in CLI (the vault plugin's
login skill).
Without the CLI, hand the strings to the developer to run first.

Report back:

- The files written, their paths, and the packages they declare.
- The entry point chosen and why, plus any safe defaults applied in Step 2, called out explicitly.
- A one-line confirmation that the loop is bulk-safe. If the generated logic could not avoid a
  per-record service call, say so plainly rather than hiding it.
- Before reporting, read the finished class once more for the deprecated calls in Step 5's table
  (`newHttpRequest(`, `.send(`, `batchSaveRecords(` or `batchDeleteRecords(` taking a bare list,
  `query(` taking a String, `escape(`) and for `import com.veeva.vault.sdk.api.logging.LogService`,
  which does not exist. Any hit gets rewritten to the replacement, not annotated.
- Any attribute or interface detail taken from the SDK `.jar` or docs MCP rather than from this
  skill, so the developer knows which parts were verified against the current release.
- Next step: deploying this class is a separate job. In a Vault CLI workspace it goes up with the
  rest of the workspace through the vault-administrator plugin's config and mdl skills; to move it
  between Vaults as a package, see "Getting code onto a Vault". This
  skill builds no VPK and writes nothing to a Vault.
- Each VQL string run with `vault query` and whether it passed, or the strings handed over to
  run when there was no CLI.

## Getting code onto a Vault

The same answer covers a class written here and classes being moved from one Vault to another,
for example triggers that came down in a `vault config pull`. This skill still deploys nothing.
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` has the CLI's commands and rules.

- In a Vault CLI workspace, the classes under `javasdk/` deploy with `vault config push`, in the
  same push as the components that name them. Vault compiles the source on deploy, so the
  developer needs no local SDK jar, Maven build, or compile step to deploy, and a session never
  needs to search the disk for one. The loop is the workspace loop the vault-administrator
  plugin's config and mdl skills describe: pull, edit, preview, push.
- To move code between Vaults as a package, or to deploy without a workspace, use a VPK that
  carries the `javasdk` folder, exported from the source Vault or built by hand. The two calls are
  Import Package (`PUT /api/{version}/services/package`, the `.vpk` as the file) and Deploy Package
  (`POST /api/{version}/vobject/vault_package__v/{package_id}/actions/deploy`), through
  `vault api` or with the session the user already has.
  A Maven project changes nothing here:
  Vault compiles on deploy, so the project's build is not part of deploying. Never route the
  deploy through a build-tool plugin that reads its own credential file, and never ask the user
  to put a password, session ID, or token into a project file; the signed-in session is the
  only credential a deploy needs.
- Deploying the class is the registration. `@RecordTriggerInfo` (and the matching annotation
  on an action or a job processor) tells Vault which object and events the class runs on, and
  the trigger is live once the deploy succeeds. There is no separate step to register it, in
  MDL or in the UI. Admin > Configuration > Record Triggers lists what is deployed and whether
  it is enabled, which is where to confirm it.
- A pulled workspace does not show whether a class is deployed. `Recordtrigger`,
  `Recordaction`, and the other code component types are class `code`: MDL cannot create or
  describe them, and `vault config pull` does not write them under `components/`. Finding no
  `Recordtrigger` folder means nothing. To see what is deployed on a Vault, query it:
  `SELECT component_name__v, status__v FROM vault_component__v WHERE component_type__v =
  'Recordtrigger'`. Never tell a developer their code is unregistered or not running on the
  strength of the workspace alone.
- Order matters when code and configuration travel together. A trigger names its object, so the
  object has to exist on the target first; an MDL component that names a class (an
  `Objectaction` pointing at a `Recordaction`, say) needs the class deployed first.
- Before any of it, check the classes against the target: the same objects and fields they
  read and write, by API name. A class that compiles against the source can fail validation on
  a Vault where a field is missing.
- The target's classification comes from the vault plugin's login skill, and it goes in the
  hand-off with the target DNS, the same as for any write.
