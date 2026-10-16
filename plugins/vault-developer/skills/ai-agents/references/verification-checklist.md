# Verification checklist

Run every check that applies to what was just generated. This is the load-bearing subset, not the
exhaustive one: it catches the failures that are silent, where the import succeeds and the behavior
is wrong. Fix in place and re-check; do not report an artifact done with an open item.

The general Vault Java SDK checks are not repeated here. The JDK allowlist, `VaultCollections`,
broad `catch`, regex and `String.format`, `RollbackException` as the only throwable, boxed numbers
into `Object` parameters, picklist values as a list, and bulk-safe service calls all belong to the
java-sdk skill's `references/platform-constraints.md`. Run those first, then this file.

## Every Java file here

- [ ] `@ExecuteAs` and `ExecuteAsUser` import from `com.veeva.vault.sdk.api.executeas`; the AI and
      agent types from `...api.ai` and `...api.agent`. A wrong package compiles and fails import.
- [ ] The handler-specific annotation sits alongside `@ExecuteAs`. Missing it is the usual cause of
      an import failing with "Implementing interface [...] is not allowed".

## Context type handler

- [ ] `AiScope` in the annotation, `AiScopeType` in every `getScopeSource(...)` call, and the
      returned variable's type all name the same scope. Skip only if `getScopeSource` is never
      called.
- [ ] `onResolve` returns through `newSuccessResponseBuilder()`, appending at least one output item.
- [ ] A `JsonObject` returned through `withText(...)` is serialized with `.asString()` first.
- [ ] Any claim to read document text goes through `HttpService` and a named Connection to the
      document text endpoint. No invented `getText(...)` or content field; the SDK has neither. Any
      version-id parsing uses `indexOf` plus `substring`.

## Tool type handler

- [ ] `dynamicToolSpec` is stated explicitly, never left to the default. `true` implies
      `onGenerateInput` is implemented and returns a built response; `false` implies it is absent.
- [ ] `JsonResultBuilder.class` pairs with `.withJson(...)`, `TextResultBuilder.class` with
      `.withText(...)`. Never mixed.
- [ ] Every node built through the dynamic builders carries a non-empty description.
- [ ] Both `Aitooltype.<name>.mdl` and `Aitool.<name>.mdl` exist, and the `Aitool`'s `tool_type(...)`
      names the type just emitted. The `Aitooltype` alone is an unfinished artifact.

## AgentActionResultHandler

- [ ] `@AgentActionResultHandlerInfo` present, class in its own file, both `onSuccess` and `onError`
      implemented.
- [ ] `onSuccess` references nothing captured from outside the handler. It runs in a new
      transaction, so everything comes from `result.getScopeSource(...)`.
- [ ] Document scope reads `version_id` as `ValueType.STRING` rather than reassembling it from the
      numeric version fields.
- [ ] `onError` logs and returns. No write-back, no throw.

## Record action or trigger that calls an agent

- [ ] The instance is stopped after the last `runAgentAction(...)` for it, and one instance covers
      the actions in the same unit of work rather than one instance per action.
- [ ] Trigger only: `AFTER_INSERT` and `AFTER_UPDATE` only, never a `BEFORE_*` event.
- [ ] Trigger only: every `change.getOld()` sits inside a branch gated on the firing event.
- [ ] Trigger only: it iterates `getRecordChanges()`, not `getRecords()`.
- [ ] Trigger only: a loop guard is present when the write-back target is the originating record or
      the same object, and absent when it is a different object.
- [ ] Trigger only: the start call is skipped on an empty request list, and nothing is saved inside
      the trigger's own `execute(...)`.
- [ ] An existing handler matching this agent, scope, and write-back field is reused rather than
      duplicated.

## Every MDL file

- [ ] Every custom component and subcomponent name ends in `__c`; a referenced standard type keeps
      its `__sys` suffix.
- [ ] `runtime_code(...)` uses the literal `Aicontexttyperuntimecode.` or `Aitooltyperuntimecode.`
      prefix plus the fully qualified class name.

## Aicontexttype (custom)

- [ ] `admin_configurable(true)` present, no `configuration_code(...)` clause, and every
      `Agentcontext` referencing this type uses empty `context_configuration()`. The component
      reference marks `admin_configurable` read-only even though the platform's own example sets it,
      so a rejection on that clause is not a fault in the file.

## Aitool (static mode)

- [ ] Every `properties.<key>` and every `items` node has a non-empty `description`, nested nodes
      included. Fill one in where the user gave enough to write it, and report it; otherwise write
      a `TODO` description and flag each one. Never ship a node with no description.
- [ ] `input(...)` is present even on a zero-parameter tool
      (`{"type":"object","properties":{},"required":[]}`). Dynamic mode is the only case that omits
      it.
- [ ] `description('...')` is present and 1024 characters or fewer.
- [ ] Standard-tool wrap: every runtime token inside `configuration({...})` is brace-escaped,
      closing brace included, and tokens and input properties match with no orphans either
      direction.
- [ ] VQL Query wrap: no `LIMIT`, `PAGESIZE`, or `PAGEOFFSET`; an object or document target; ten
      parameters or fewer; an `ORDER BY` on anything returning several rows; an `AS` alias on every
      function call and every `id`. The query text itself is the query skill's review, not this one's.
- [ ] JSON Response wrap: the description says what the tool reports, not that it outputs JSON.

## Agent

- [ ] `class(...)` matches the scope, with the unused one of `document_type` / `object` left as
      empty parens, and the populated one carrying its literal type prefix.
- [ ] `agent_user('[agent]<api_name>__c')`: literal `[agent]` prefix present.
- [ ] `model_level(...)` is `BASIC` or `ADVANCED`.
- [ ] `description(...)` on the Agent and on every `Agentaction` is present and within 500
      characters, and `support_chat` and `tool_evaluation` are stated on every action.
- [ ] `tool_evaluation('REQUIRED')` has a populated `tool_evaluation_required_tool(...)` whose tool
      is also wired through an `Agentactiontool` peer.
- [ ] Every action the user said will be invoked over the Vault API or MCP has `api_access(true)`.
- [ ] Every tool and context an action uses has its peer wiring, except a context that is
      `always_include(true)`. Those have no `Agentactioncontext`; `always_include(false)` ones have
      at least one.
- [ ] A custom context type's `Agentcontext` has empty `context_configuration()`; a standard one has
      the XML envelope populated and brace-escaped.
- [ ] A tool whose input takes a scope id has a context binding surfacing that id.
- [ ] Every `instructions(...)` body names its data sources, carries a Markdown output template, and
      ends with a negative-rule list including Markdown only and no raw HTML. No literal HTML tags
      inside the body; name the tags in words.
- [ ] A count or total in the answer matches a tool result or a VQL run alongside it, and the
      instructions only ask for one when a tool returns it.
- [ ] `hide_in_chat` is empty or `false`.
- [ ] The block came from a verbatim retrieval of the live agent if it already exists on the target
      Vault. `RECREATE` deletes anything the file leaves out.

## What the report may and may not claim

The checks above are static. Keep the final report inside what was actually checked.

- [ ] An undeployed file is reported as written and statically checked, never as working. This skill
      deploys and executes nothing.
- [ ] A clean import proves the syntax of a `context_configuration(...)` or a Rowset predicate,
      because Vault validates it at import time. It does not prove the predicate discriminates: a
      real filter and a no-op filter produce identical output when every related record already
      satisfies it. Only watching an excluded record disappear proves the filter works.
- [ ] Proving a filter that way needs contrasting data, and creating or toggling a record to set it
      up is a data change, not a read. Ask for explicit authorization naming the record, and restore
      the original state afterward. "Test it" does not by itself authorize changing a record.
- [ ] Testing by sending a bare user message routes through the platform's routing agent, which
      often picks a standard agent rather than the custom one. A correct-looking answer there does
      not prove the custom agent's context binding works; the LLM may simply be reading the document
      body. Force the direct form, naming the agent and the action, before concluding anything about
      a specific agent.
