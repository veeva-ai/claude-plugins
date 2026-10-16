---
name: custom-pages
version: "0.1.0"
description: >
  Scaffolds Veeva Vault Custom Pages: the Page MDL component, Java PageController, and client
  bundle. Use when a developer needs custom UI in Vault that standard object layouts or tabs
  can't provide, needs to pass data between a Java PageController and a React component, or is
  debugging shadow DOM styling or a distribution manifest. Produces code and MDL only. Never deploys, never calls the Vault API directly.
---
# Vault Custom Pages

A Custom Page is three files that have to agree with each other: a `Page` MDL component, a Java
`PageController`, and a client bundle exporting a `definePage()` call. This skill scaffolds all
three. Custom Pages are how you put genuinely custom UI in Vault, rendered inside a shadow DOM and
fed by server-side Java SDK code. Drafting only. Nothing here deploys or holds a credential, and
the one call to a Vault is a read-only run of the controller's VQL through `vault query` (step 4).

Read `references/scaffold-template.md` before generating any file, and
`references/gotchas-and-type-mapping.md` before rendering anything styled or naming anything. For
a dashboard, read `references/dashboard-design.md` too.

## Scope

Out of scope, with the skill that owns each:

- Java record triggers, the lifecycle hooks that fire automatically on record insert, update, or
  delete: the java-sdk skill.
- AI Tools, Agents, and AgentActionResultHandlers, even when the ask is an Aitool embedded in a
  page: the ai-agents skill.
- Deploying the page to a Vault: the vault-administrator plugin's config and mdl skills for a
  workspace push. Moving the page between Vaults takes a VPK that carries the `websdk/` layout
  for the client bundle, exported from the source Vault or built by hand, imported and deployed
  with Import Package and Deploy Package through `vault api` or the signed-in session.
- Vault UI SDK app pages and controls (`ControlRegistry`, `PageControllerContext`,
  `@vault/uisdk`), a different and older mechanism. Its MDL attributes (`default_client_code`,
  `server_code`) must never be set on a Custom Page's `Page` component.

## Handoffs

- VQL query construction, meaning syntax, quoting, relationship traversal, and operators: the
  vault plugin's query skill, loaded for every query the controller runs (step 3).
- MDL syntax and component conventions beyond the `Page` attributes in step 3: the
  vault-administrator plugin's mdl skill.
- SDK-wide data types (Vault field type to `ValueType`, what `JsonObjectBuilder.setValue` accepts)
  and the `QueryService` call shape: the java-sdk skill's `references/platform-constraints.md`.
- Vault API endpoints, request and response shapes, auth, and rate limits: verify against
  the Vault docs MCP rather than asserting from memory.

## Default stack

Framework, language, and bundler are the developer's choice, and bundling is recommended rather
than required. Default to `references/react-esbuild-example.md`, which works React, plain
JavaScript, and esbuild end to end; read it
for the project layout, npm install steps, `esbuild.mjs` config, and which built file the manifest
points at. Match an existing client layout if the project has one, and use whatever other framework
or bundler the developer names: the pieces a Custom Page is made of do not change with the stack.

## Step 1: Parse the Input

Work out what the developer actually needs. Match it to one of these jobs:

- **New Custom Page from scratch**: all three pieces (MDL, controller, client bundle)
- **Client-only page**: no server data yet, so the controller can wait or be skipped. A `Page`
  component still requires `page_controller` once `client_distribution` and `page_client_code`
  are both set, so flag that if the developer wants no controller at all
- **Add/modify a `PageController`**: a server-side data fetch, an `onEvent` handler for
  client-triggered calls, or a data-shape fix
- **Add/modify client code**: a component, a `sendEvent` call, or broken styling
- **Fix broken styling**: nearly always the Shadow DOM boundary. Read that section of
  `references/gotchas-and-type-mapping.md` first
- **Build the client distribution**: `distribution-manifest.json` plus the built bundle, written
  into the workspace's `custom_pages/{distribution_name}/` folder

One request usually names more than one job. "Give me a page that shows the current user's active
tasks" is a new page plus a controller that reads data plus client rendering. Resolve every job
named. Don't collapse them into a stub.

## Step 2: Gather Missing Information

Ask only for what's missing and actually needed to draft the code.

Naming comes first: the page's label, its `__c` API name (derive it from the label if the
developer doesn't give one), and the `url_path_name` slug, URL-safe and 40 characters max. If
the project already has a Java package convention, put the controller in it; otherwise draft under
`com.veeva.vault.custom.pages` and say that's what you did.

Then the data question, which is really two. Does the page need server code at all? Static or
client-rendered content can get by with a minimal pass-through controller, but a `Page` with
`client_distribution` and `page_client_code` set still requires `page_controller`, and the
reference file's MDL section has the smallest controller that satisfies it. And what crosses the
Java/client boundary, from where? Vault data is read using VQL, and the only page-specific part is
the round trip: the controller runs the query, puts the result into the load response, and the
client receives it as a `JsonObject`. Get the object and the fields, then hand the query
construction itself to the vault plugin's query skill rather than composing VQL here (step 3).
Another SDK service crossing the same boundary works the same way. A direct client-side API client
also exists for the exception case where a page must reach Vault without a controller round trip;
its shape is in `references/scaffold-template.md`; verify endpoint detail against the Vault
docs MCP.

The rest are quick. No CSS framework by default. Tailwind, MUI, Chakra, and styled-components all
work but each needs the Shadow DOM wiring from the appendix of
`references/gotchas-and-type-mapping.md`, so raise it only if the request implies a UI library.

Never ask for a session ID, API key, or password. This skill produces code and commands. It does
not run them, apart from the read-only query check in step 4.

## Step 3: Generate the Files

Draft each piece against the shapes in `references/scaffold-template.md`, checking anything styled
or named against `references/gotchas-and-type-mapping.md`:

- **MDL**: a `RECREATE Page {name}__c (...)` block with `label`, `active`, `client_distribution`,
  `page_client_code`, `page_controller`, and `url_path_name`. Use single-quoted attribute values.
  Never set `default_client_code` or `server_code` on a Custom Page: those are for a different,
  older page mechanism, and Vault docs mark them internal-use-only on this component. Those
  attributes are the set this skill states from memory. For any optional `Page` attribute, or any
  attribute or behavior that isn't already familiar, look it up with the docs MCP
  (`mcp__vault-docs-mcp__search_documentation`) before asserting it.
- **Java `PageController`**: `@PageControllerInfo` + `@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)` on
  the class, `onLoad(PageLoadContext)` returning a `PageLoadResponse` via
  `context.newLoadResponseBuilder().withData(...).build()`. Add `onEvent(PageEventContext)` only
  if the client needs to push data back mid-session (`sendEvent` from the client side). Use
  `ServiceLocator.locate(...)` for every SDK service, never `new`. Use `VaultCollections.new*`
  for any collection the SDK needs to see, never a raw `java.util` constructor. Load the java-sdk
  skill before writing the controller: its query rules (a literal list matched with `CONTAINS`,
  `onError` on every query reporting `getMessage()`) apply to every query in it.

  Every query in the controller goes through these steps, even a one-field `SELECT`:

  1. Load the vault plugin's query skill and get the `SELECT` from it, using the object and
     fields from step 2. Don't write the VQL here, however simple it looks.
  2. Put that exact string in the controller.
  3. Run it in step 4 before handing anything over.
- **Client bundle**: a `definePage(({ data, element, pageContext, sendEvent }) => {...})` entry
  point and a `distribution-manifest.json` pointing each page at its built entry point, never the
  source file.
- Apply the Shadow DOM styling rule from the reference file to any styled output: list the built
  CSS in the manifest's `stylesheets` array first, and use a `<style>` block as a JSX child only
  for CSS that has to be generated or conditional. Never a `document.head` injection or an
  external `<link>`.
- Apply the type rules from the java-sdk skill's `references/platform-constraints.md` to any value
  crossing the Java-to-JSON boundary or read from a query result, and check the controller against
  that file's "Forbidden constructs" table. A local Maven build checks none of it; Vault's deploy
  validation does, so `new RuntimeException(...)` in a query's `onError` compiles fine and then
  blocks the package. Throw `RollbackException` there.
- In a Vault CLI workspace, the built bundle and `distribution-manifest.json` go in
  `custom_pages/{distribution_name}/`, and nothing else does. The CLI zips that whole folder as the
  distribution, so client source, `package.json`, and `node_modules` live in a project folder
  outside the workspace and build into it. Give the build command as a reference command and say
  plainly that it is not being run.
- For a dashboard, follow `references/dashboard-design.md`, including the cube-shaped controller
  payload when the page filters in the browser.
- If the request only loosely maps onto a Custom Page, and what it really wants is an object page
  layout change, a Vault UI SDK control, or a trigger, say so and point at the right tool rather
  than forcing a Custom Page scaffold.

## Step 4: Confirm

Before reporting, run each `SELECT` from the controller once, exactly as written, read-only
against the target Vault, with real values in place of any token:

```bash
vault query "<VQL>" --dns <dns>
```

It should come back with rows. If it errors, fix it through the query skill, update the
controller, and run it again. Don't hand over a controller whose query fails. An empty result on
an object with no records yet is not a failure, but say so. Without the CLI or a reachable Vault,
hand the strings to the developer to run before the deploy and say plainly that the queries are
unverified.

Report back:

- Every file drafted (MDL, `.java`, the client entry point, `distribution-manifest.json`, the
  build config as applicable), each with its intended path and a one-line purpose.
- The API names used: `{name}__c` for the page, the matching name in the distribution manifest,
  and the MDL `client_distribution`/`page_client_code` references. All three have to match
  character for character or the page simply won't resolve at deploy time.
- Any `ValueType`/`JsonValueType` conversion applied and why (name the source Vault field type).
- Whether the Shadow DOM styling rule was applied, and to which file.
- Each `SELECT` in the controller and whether it ran and returned rows, or is unverified.
- **Next step is the workspace loop.** In a Vault CLI workspace the built client sits under
  `custom_pages/{distribution_name}/`, the `PageController` under `javasdk/` by package path, and
  the `Page` MDL under `components/`. Deployment goes through the vault-administrator plugin's
  config and mdl skills, not through a hand-built upload or a separate move per piece. To move
  the page between Vaults as a package, one VPK holds the `Page` MDL,
  the controller under `javasdk/`, and the built client under `websdk/`, imported and deployed with
  Import Package and Deploy Package.
  This skill runs none of it.
- **How users reach the page**, as a question with the options, unless the request already says.
  Do not pick one:
  - The URL only, `https://{HOST}/ui/#custom/page/{url_path_name}`, shared or linked from wherever
    the developer wants. Nothing more to draft.
  - A `Tab` in the navigation bar, or a `Subtab` under an existing tab. Users need View on the tab
    as well as the page.
  - A `Pagelink` on an object, which sends users to the page in place of that object's record
    View, Edit, Copy, or Create action.

  Whichever they choose, users need View on the `Page` in a permission set, under
  Admin > Users & Groups > Permission Sets > Pages. A Tab, Subtab, or Pagelink names the `Page`,
  so it goes in a second push after the `Page` has landed; a Tab in the same push is refused with
  "value [Page.<name>] does not resolve to a valid [component]".
- Anything marked `~` in the reference material that applies to this page. Don't bury a caveat
  that decides whether the drafted code works once deployed.
