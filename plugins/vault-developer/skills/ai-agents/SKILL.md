---
name: ai-agents
version: "0.1.0"
description: >
  For a developer building a Vault AI Agent: the agent definition, the tools and contexts it
  calls, and the Java SDK handlers behind them. Use when scaffolding an AI Tool Type or Context
  Type runtime handler, an AgentActionResultHandler, a record action or trigger that runs an
  agent and writes the result back, an Aitool wrapping a standard tool type, or a new Agent's
  MDL wiring.
---

# Vault AI Agents

Vault AI is a separately licensed feature, so treat the artifacts here as available only on a
Vault that has it enabled. This skill writes the `.java` and `.mdl` files that make up an agent
and hands them back for the developer to deploy. It holds no credentials and deploys nothing. The
one call it makes to a Vault is a read-only run of a handler's VQL through `vault query` (Step 4).

## Before anything is deployed

Confirm Vault AI is on for the target Vault: the Vault AI settings under Admin > Settings, or the
developer's word for it. One read answers it from the API: `vault api /services/ai/agents`
returns `OPERATION_NOT_ALLOWED` with a message to enable the Veeva AI feature flag when it is off,
and a list of agents when it is on. Delivered `__sys` agents and tool types exist whether or not it is on,
so finding them proves nothing. With it off, `Aitooltype` and its Java handler still deploy, but
`Aitool` and `Agent` fail on import with `GEN-FREC-II-1036 Unknown component [Aitool]` or
`[Agent]`. That error means Vault AI is off, not that the MDL has a typo.

The workspace commands are in `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`, and the loop is the
vault-administrator plugin's config and mdl skills. Start from those, so a session that begins
here does not rediscover the CLI or look for it on disk. If a command fails or behaves differently
from the reference, check its `--help` and follow the installed CLI.

The surface is small and unforgiving. A handler missing its `@*RuntimeHandlerInfo` annotation
imports as a plain class Vault then ignores, with nothing to say why the agent never calls it.
MDL that drops a required attribute or the `__c` suffix at least fails loudly. Ground every
generated file in the reference files rather than guessing at an import path or an attribute name.

## Scope

What this skill owns: the artifacts that are specific to an AI Agent. The `Agent` block and its
`Agentaction` / `Agentactiontool` / `Agentcontext` / `Agentactioncontext` wiring, the
`Aicontexttype` and `Aitooltype` / `Aitool` components, the Java runtime handlers those components
point at, the result handler that writes an action's output back, and the record action or trigger
that starts an agent instance.

Out of bounds, with the skill that owns each:

- MDL as a language: statement forms, ALTER versus RECREATE, the execute endpoints, the validator
  failure modes, and how to read an execution response. The vault-administrator plugin's mdl skill.
  This skill knows what an `Agent` block has to contain; that one knows how MDL behaves.
- Java SDK platform constraints: the JDK allowlist, `VaultCollections`, `LogService`, the
  restrictions checked at import, governor limits, and bulk-safe service-call patterns. The
  java-sdk skill, `references/platform-constraints.md`. Those rules apply to every handler written
  here; this skill does not restate them.
- A record trigger, record action, job, or custom API with no agent in it. The java-sdk skill.
- The VQL text inside a VQL Query tool. The vault plugin's query skill. This skill
  shapes the wrapper and the parameter tokens; that one shapes the query.
- Running an agent action over the Vault API, reading its execution status, retrieving a deployed component
  as MDL, or resolving a doctype or object API name. The api skill.
- Signing in, credentials, and which Vault is on the other end. The vault plugin's
  login skill.
- Packaging and deploying the finished files. The vault-administrator plugin's config and mdl skills
  push them within a Vault; a package moves them between Vaults.
- The Custom Page three-file shape. The custom-pages skill.

## Handoffs

Name the owning skill and stop at the boundary rather than approximating its job. That holds
however the ask is worded, including "just draft the MDL command" or a paste of a half-written
execute call to finish.

## Step 1: Identify the artifact

Match the request to one of these seven jobs. A request can name more than one: an action that
summarizes a record and writes it back is a record action plus its result handler together.
Resolve every job named, do not collapse them silently.

- **Custom AI Context Type handler**: `AiContextTypeRuntimeHandler`, surfaces data to the LLM.
  Most agents do not need one; check the standard context types in `references/mdl-shapes.md`
  first.
- **Custom AI Tool Type handler**: `AiToolTypeRuntimeHandler`, static or dynamic input spec.
  Always paired with a concrete `Aitool`. An `Aitooltype` alone is not callable.
- **Aitool wrapping a standard tool type**: no Java, one MDL file. VQL Query and JSON Response are
  the documented standard tool types.
- **AgentActionResultHandler**: runs in its own transaction once an agent action completes and
  writes the output somewhere. Rarely generated alone; it is the second half of the two entries
  below.
- **Record action that starts an agent and writes the result back**: a user clicks a button, the
  action starts an agent instance against that record, runs an action, and the paired result
  handler drops the output onto a field.
- **Record trigger that starts an agent and writes the result back**: the automatic version,
  firing on `AFTER_INSERT` / `AFTER_UPDATE` instead of a click. Same result-handler shape, extra
  hygiene rules (event gating, loop guard).
- **Define a new Agent (MDL only)**: a top-level `Agent` block plus one or more `Agentaction`
  children and the peer wiring subcomponents. This registers the agent and connects it to tools
  and contexts that already exist, or that the user names for a later run of this skill.

If the request is vague, ask which of the seven it is before proceeding. Guessing here generates
the wrong artifact entirely.

## Step 2: Gather the inputs

Ask only what the identified artifact needs, in a single message, and do not interrogate for
fields a sensible default covers.

- **Every artifact**: API name (component names need the `__c` suffix), display label, and for
  Java artifacts a class name plus package.
- **Context type**: scope (`DOCUMENT`, `OBJECT`, or `ALL`), what the context should expose, and
  whether it needs per-binding parameters. A custom context type cannot take them; see the
  workarounds in `references/mdl-shapes.md`. Ask first whether the data is tagged on or linked to
  the scoped document or record, because that is a relationship reachable from a standard context
  rather than a reason to write Java. Ask separately whether the context needs extracted document
  text; the SDK exposes no such API and the Vault API fallback needs the user's agreement.
- **Tool type**: static or dynamic spec, result type (`JSON` or `TEXT`), what the tool does, and
  for static mode the input JSON Schema. Convert the user's plain-language description yourself,
  with a non-empty `description` on every property.
- **Standard-tool Aitool**: which standard tool type, the LLM-facing description, the query or
  schema body, and the input parameters (name, type, description, required).
- **Record action or trigger**: object API name, agent configuration name plus agent action name,
  the agent's scope, the target field and type the result gets written to, the result format, and
  the prompt sent to the agent. For a trigger: which events, and whether the write-back lands on
  the originating record, another record on the same object, or a different object. That last
  answer decides whether a loop guard is required.
- **New agent**: objective, scope (`DOCUMENT` plus doctype, or `OBJECT` plus object), model level
  (`BASIC` or `ADVANCED`), and per action a label, an output shape, what data it needs, which tools
  it calls, and whether the action will be invoked directly over the Vault API.

Object and doctype API names carry a namespace suffix. If the user names one loosely, ask rather
than guessing the suffix, or hand the lookup to the api skill.

## Step 3: Generate the scaffold

1. **Load the java-sdk skill** before writing any handler. Its platform rules and its query rules
   (a literal list matched with `CONTAINS`, `onError` on every query, each query run before
   deploy) apply to every handler here, and reading its reference files alone skips them.
2. **Inspect the project** before writing Java: find the source root and the MDL resources root,
   read one existing handler and one existing MDL file if any are there, and match their package
   naming and directory convention. If no project structure turns up, ask where the files go.
3. **Read the matching section** of `references/handler-scaffolds.md` for Java or
   `references/mdl-shapes.md` for MDL. Do not skip this even for a familiar shape; the annotation
   and attribute rules are exact, not approximate.
4. **Write the files**, following the pairing rules in those references:
   - A custom tool type always ships two MDL files, `Aitooltype` and `Aitool`, never one alone.
   - A record action or trigger always ships two Java files, the entry point and its paired result
     handler, unless an existing handler already matches the agent, scope, and write-back field.
     Reuse it in that case.
   - A new agent ships one MDL file holding the `Agent` block and every peer it needs.
   - A tool with an optional mode (a scope, a filter, a level) never defaults to one mode
     silently, because the user never learns the others exist. Either the tool returns every mode
     in one call and the answer shows them all, or the action asks through the Ask User agent
     tool, which runs only on an action with `support_chat(true)` and has to be described in the
     action's instructions. The action's description says which of the two it does. Vault Help's
     "Configuring Agent Tools" page is the source for Ask User: it names the tool, says it is
     system-managed and assigned to an action, and says it runs only with Support Chat on.
5. **Add a short doc comment** at the top of each Java class stating the purpose in the
   user's own terms. It is the only documentation the class carries once deployed. MDL has no
   comment syntax, so an MDL file holds the statement and nothing else; a comment in it fails the
   parse on push. Four-space indentation, one artifact per directory, and no `README.md` unless
   the user asks for one.
6. **If the component already exists on the target Vault**, start from its current MDL rather than
   hand-authoring a partial block. `RECREATE` replaces the whole component and drops anything the
   file leaves out. In a workspace, the pull at the start of the change brings the component
   files down; the config skill owns that freshness check, so it is not repeated here. Without a
   workspace, the retrieval is the api skill's call.

## Step 4: Self-verify

Open `references/verification-checklist.md` and run every check that applies to what was just
generated. Fix in place and re-check. Never report success with an open item.

Then run every VQL string in a handler read-only with `vault query "<VQL>" --dns <dns>`, with
real values in place of any token, and fix the ones that fail before the hand-off. A query that fails inside a
deployed handler costs a deploy and a debug log to find; here it costs seconds. Without the CLI,
hand the strings to the developer to run first.

This step earns its keep more than in most skills. The generated code runs unattended, and bad
MDL fails import with a validator error the developer then has to trace back by hand.

## Step 5: Confirm

Report back:

- The file paths written and a one-line summary of each. No full file bodies unless asked.
- The checklist items run and their pass or fail, so the developer can tell what was checked from
  what was assumed.
- Wiring still left to do by hand: an `Agentaction` pointing at a tool or context that does not
  exist yet, or the `Objectaction` that registers a generated record action as a user-visible
  action. Both of those are MDL authoring; hand them to the vault-administrator plugin's mdl skill.
- Any `~` claim from the reference material that bears on whether the file will import.
- Where a handler's `.java` file goes, and how it reaches a Vault, is the java-sdk skill's
  "Getting code onto a Vault": under `javasdk/` by package path in a Vault CLI workspace, and up
  with `vault config push`. Vault compiles the source on deploy, so no local SDK jar or compile
  step is needed. The agent, tool, and context MDL files go in the same workspace, under
  `components/`, and deploy through the same workspace loop; they are not sent as direct
  `RECREATE` statements. They depend on each other and on the handler classes (`Aitooltype`, then
  `Aitool`, then `Agent`), and a push does not order them, so the mdl skill's plan stop handles
  that order. Commit the workspace before any pull, since a pull can remove files that have not
  been pushed.
- Route the next step, do not take it. Deploying belongs to the vault-administrator plugin's
  config and mdl skills.
  Calling the Vault
  API to resolve an API name, check a status, or retrieve a deployed agent belongs to the api skill.
- Where users reach the agent, and how to diagnose a failed run, are in
  `references/running-and-troubleshooting.md`. Put the first in the hand-off, in place of "open
  the AI tab". An agent scoped to an object is tested by opening a record of that object and using
  the AI Chat icon on its page. Do not flip `api_access` on to test it through the API unless the
  user asks for an API test.
