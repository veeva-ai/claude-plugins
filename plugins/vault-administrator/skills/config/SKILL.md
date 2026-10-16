---
name: config
version: "0.1.0"
description: >
  For an admin, a developer, or another skill that needs to know how a Vault is configured:
  its objects and fields, its lifecycles and workflows, and what depends on what. Use it to pull a Vault's configuration into a local workspace once, keep that workspace
  fresh, and answer configuration questions from the files instead of calling the API
  component by component.
---

# Vault Config

A Vault's configuration changes slowly and gets read constantly. This skill pulls it once
into a workspace on disk with the Vault CLI's `vault config pull`, and points every other
skill at the files. Without the CLI there is no workspace, and the skill answers the
question in front of it from the API instead.

`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` is the CLI surface every skill in these
plugins assumes: the commands, the rules for using them, and the known limits. Read it
before the first CLI command in a task.

## Scope

Pulling configuration and answering questions about it from the workspace. Out of bounds:

- Changing configuration. Authoring or running Metadata Definition Language (MDL) is the mdl
  skill, and so is the edit and push loop in a workspace. Packaging is a package built by hand or
  in the Vault UI (Admin > Deployment > Outbound and Inbound Packages).
- Record counts and document questions go to the vault plugin's query skill.
- Admin > Settings. They are not components, so they are not in the pull. Point at Admin >
  Settings in the UI and say so.

## Handoffs

Sign-in comes from the vault plugin's login skill; do not reason about credentials here. Once the workspace
exists, hand the question to the skill that owns the work, with the file path that answers
it. Changes go to mdl, queries to the vault plugin's query skill, anything under `javasdk/` to
java-sdk, and anything under `custom_pages/` to custom-pages.

## Step 1: Check for the CLI

```bash
vault version
```

An answer of 26.2.4 or later means the CLI path, Steps 2 to 4. No answer, or older than 26.2.4,
means offer to install it first, per rule 3 in `reference/vault-cli.md`, and run the install
only on the user's yes. If they say no or it fails, Step 5, and do not raise it again.

## Step 2: Find or create the workspace

A workspace is any directory with `.vault/state.json`. Look in the directory the user names,
then the current project (its root or its `vault-config/`), and nowhere else. Workspaces under
the home directory or in other projects may belong to other Vaults; use one only when the user
names it. Before trusting what is there, check which Vault it came from:

```bash
grep -o '"__source_dns__": *"[^"]*"' <workspace>/.vault/state.json
```

`state.json` holds a hash for every tracked file and runs to megabytes, so search it and
never read it whole. A workspace pulled from some other Vault does not become right by
refreshing it; use a different directory.

None found: pull one. When the user names a Vault ("my QA sandbox", "DEV"), resolve the name
the way login does: ask for the DNS when they gave only a name, pass it with `--dns`, and say
which DNS you used. A guessed label fails, a
guessed DNS can land on the wrong Vault, and another signed-in Vault is not a stand-in for it.

The new workspace goes in the directory the user names. If they name none, it goes in
`vault-config/` under the current project, so the files sit in the project they are working
in, and you say the path in one line. Don't put it under the home directory unless they ask
for that. If `vault-config/` already holds a workspace from another Vault, ask where the new
one should go. Then:

```bash
vault config pull --dns mycompany-qa.veevavault.com --dir vault-config
git -C vault-config init -q && git -C vault-config add -A && git -C vault-config commit -qm "first pull"
```

A pull is a read against Vault. It runs on any Vault, production included, and writes only
inside the workspace directory. Inside that directory it writes over any file you changed
without asking, and stops only before removing files Vault does not have, which includes the
files of a new component you have not pushed yet. So the
workspace is a git repository from its first pull (`git init`, then commit), and every pull
starts with a commit. Never add `--yes`, except the committed-workspace recovery in rule 5 of
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`.

After upgrading the CLI, pull again before the first push. A push from a workspace pulled
with an older CLI has proposed deleting components the workspace did not hold.

Keep diagnostics beside the workspace, not inside it; `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`
says why.

## Step 3: Check it is current

Before answering from an existing workspace:

```bash
vault config pull --dry-run --dns mycompany-qa.veevavault.com --dir <workspace>
```

A dry run against the Vault the workspace was pulled from is the freshness check. It writes
nothing. For components, Java classes and custom pages it prints a count line such as
`2 updated, 498 unchanged` at the end, after one line per item a pull would change (`MODIFIED`, `ADDED`,
`DELETED`, `ERROR`). A file you edited shows as `MODIFIED` too, since a pull would write over
it. Only unchanged counts means the workspace is current. An `ERROR` line whose message reads
`Failed to fetch component <Type>.<name>: Vault did not return an MDL definition, it returned 'N/A'.`
or `Vault returned an empty MDL definition.` is a component Vault does not return as MDL. It
errors on every pull and does not make the workspace stale. A few changes: pull, and Step 2's rule about what a pull removes
applies. Most of the workspace changing means it came from an older pull or a different tool, and
a pull replaces it. On a full workspace the dry run takes a couple of minutes; run it in the
background and wait for it.

Run it once per session, the first time you answer from the workspace, and tell the user what it
reported. A dry run makes the same API calls as a pull, so repeating it for every question buys
nothing. After that, answer from the files and say the workspace is as of that check. Run it
again only when the user asks; a change starts with its own pull.

Use the pull dry run for this check, not `vault config push --dry-run`. The push preview lists a
change made in Vault since the last pull as an update the push would undo, which reads like an
edit of your own.

Skip the check when the user pulled moments ago in this same session. A week-old workspace
answered without a check is a guess wearing a file path.

## Step 4: Answer from the files

`references/workspace-layout.md` maps question to file. The short version:

- An object's attributes, relationships, and lifecycle: `components/Object/<name>/<name>.mdl`.
  Its fields are split out, one folder each, under `components/Object/<name>/Field/`.
- Picklist values: `components/Picklist/<name>/`.
- States and workflows: `components/Objectlifecycle/`, `components/Doclifecycle/`,
  `components/Objectworkflow/`. A subcomponent has its own folder, and XML bodies sit beside
  its `.mdl` (`rule.xml`, `step_detail.xml`, `page_markup.xml`).
- What a component references: `.vault/dependencies.json`, a map of `<Type>.<name>` to
  `{"depends_on": [...]}`. Dependents of X are the keys whose list contains X.
- Object types: `components/Objecttype/<object>.<type>/`, separate from the object itself.
- A type's attribute vocabulary: `.vault/metadata/<Type>.json`.
- Java SDK source: `javasdk/`, by package path. Custom Pages: `custom_pages/`.

### Scoping a feature, to copy it or to describe it

"What did we build for HR" or "move the inventory setup to the other Vault" is a question
about a set of components, and the set comes from the dependency graph, never from searching
names. A picklist called `department__c` belongs to the employee feature and no keyword finds
it; a field that points at a missing picklist is the first thing to fail on the target.

1. Get the seed objects from the user, or from the custom (`__c`) objects when they say "our
   stuff". Ask which objects when it is not obvious; a shared object such as an address is
   theirs to include or leave out.
2. Walk `.vault/dependencies.json` outward from the seeds in both directions: what each
   component lists under `depends_on`, and every key whose `depends_on` names it. Keep going
   from each custom (`__c`) component that turns up until nothing new appears. Forward collects
   the picklists, lifecycles, and other objects the seeds need; backward collects the page
   layouts, validations, workflows, tabs, document fields, and doctypes that hang off them.
3. Stop at every standard (`__v`, `__sys`) component: record it as referenced, and do not walk
   through it. Standard objects such as `user__sys` connect to everything, and walking through
   one pulls in the whole Vault. A custom component can be a bridge too. A document field
   layout or a lifecycle shared with another feature will drag that feature in; when the set
   suddenly grows past what the user described, find the one component that joins the two
   features, treat it as a leaf, and tell the user it is shared.
4. Search `javasdk/` and `custom_pages/` for the seed object names. The graph does not cover
   code. A trigger on `employee__c` is part of the employee feature, and the pull's own summary
   line says how many classes it brought down.
5. Report the set grouped by component type, with counts, the code files listed separately,
   and the load order: picklists and other leaves first, objects next, then what hangs off
   them. Standard (`__v`, `__sys`) components in the walk already exist on a target of the same
   application; list them as assumed present rather than as things to create.

The file is a flat JSON map and a full one runs to hundreds of kilobytes, so walk it with a
few lines of whatever is on the machine (`node`, `python`, `jq`) rather than reading it or
chaining `grep` context flags, which miss entries whose lists run long. `cd` into
`<workspace>/.vault` first and open `dependencies.json` by its bare name: on Windows, Git Bash
turns `~` and `$HOME` into `/c/Users/...`, which Python and Node cannot open.

Do not call the set complete until steps 2 to 4 have all run. "Nothing else touches this" is a
claim about the whole graph, and only the walk can back it.

A third thing to keep straight before the two below: some component types never come down in
a pull. The code types (`Recordtrigger`, `Recordaction`, and the rest of class `code`) exist on
the Vault and are queryable in `vault_component__v`, but the pull writes the source under
`javasdk/` and nothing under `components/`. A missing folder for one of those types is not
evidence that the Vault has none. Check the Vault with the query skill before saying so.

Two things the files cannot tell you. A component being present means it is configured, not
that anyone uses it; check data with the query skill before calling a feature "in use". The same
holds for removing a field or an object: Vault refuses the delete while data is in it, the files
cannot show that, and the mdl skill takes the count at its plan stop and asks whether to inactivate instead. And custom (`__c`)
objects say nothing about which Veeva application the Vault runs; the application objects are
`__v`, and the docs MCP knows which app each belongs to.

Quote the file path with the answer so the user can open it. If the file is missing, say
which reason applies (the path is in `.vdxignore`, or the pull predates the component) and
offer the pull that fixes it. Do not fall back to the
API for a component that should be in the workspace; refresh the workspace instead, so the
next question is answered from disk too.

For a detail the MDL does not carry (an attribute's datatype, what an unfamiliar attribute
means), the type's `.vault/metadata/<Type>.json` has it, and the Vault docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_developer_documentation`) has
the component type reference.

## Step 5: Without the CLI

There is no workspace to build, so answer the question that was asked and no more. Sign-in
and the session come from login. Take the request shape from the Vault docs MCP rather than
from memory. The calls to look up are Retrieve Component Record (MDL), which returns one
component as a RECREATE statement from the unversioned `/api/mdl/components/{Type.name}`,
and Retrieve Component Record Collection, which lists the records of a type and does not
serve `Object` (objects have their own collection endpoint).

Say in the answer that it came from a live call, not a workspace, so nobody goes looking for
a file. Dependency questions ("what breaks if I remove this") need the whole graph; say that
the CLI pull is the practical way to get it, and answer what a single component's own
references show.

## When the pull fails

- A sign-in or credentials error: login's outcomes say what to do. Nothing here touches
  credentials.
- The workspace belongs to another Vault: use a different `--dir`. Do not overwrite it.
- `unknown component type`: the type name is the MDL name (`Objectlifecycle`, not "object
  lifecycle"); the component-types metadata endpoint lists them.
- `Failed to fetch component <Type>.<name>: ...`: Vault does not return that component as
  MDL. Its local file stays as it was, the rest of the pull is written, and the pull exits 1.
  It is not a CLI fault and needs no report; tell the user which components it named.
- Any other error from the CLI: check the known limits in `vault-cli.md`. If it is not
  there, capture the command, the output, `vault version`, and the Vault release for the
  user to pass to the CLI team, then answer the question through Step 5 so the task is not
  blocked.
