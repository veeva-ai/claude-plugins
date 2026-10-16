# MDL shapes

The AI Agent components this skill generates and how they reference each other. `~` marks anything the
docs do not state.

This file is not an MDL primer. Statement forms, ALTER versus RECREATE, the execute endpoints, and the
validator's failure modes belong to the vault-administrator plugin's mdl skill. What follows is only what
is specific to AI Agent components: the shapes and the traps, not the full attribute catalog. To pin an
attribute set down, retrieve a working component of that type as MDL
(`GET /api/mdl/components/{ComponentType}.{name}`, the api skill's call) or look the type up with the
docs MCP (`mcp__vault-docs-mcp__search_documentation`, source `vault_developer_documentation`).

## Component relationships

```
Agent
 |-- Agentaction (child)              one per action the agent exposes
 |-- Agentactiontool (peer)           wires one Agentaction to one Aitool
 |-- Agentcontext (peer)              wires the Agent to one Aicontexttype, with a per-binding config
 |-- Agentactioncontext (peer)        wires one Agentaction to one Agentcontext (only when that
 |                                    Agentcontext isn't always_include(true))
 |
 |-- references, by name, via 'Object.<api>' / 'Doctype.<api>': the object/doctype it's scoped to

Aitool
 |-- tool_type('Aitooltype.<name>')   references a custom Aitooltype OR a standard one; the same
                                       Aitool shape wraps either, only the tool_type value and
                                       configuration differ

Aitooltype
 |-- runtime_code('Aitooltyperuntimecode.<FQCN>')   references the Java handler class

Aicontexttype
 |-- runtime_code('Aicontexttyperuntimecode.<FQCN>')   references the Java handler class

Objectaction (registers a RecordAction as a clickable user action)
 |-- action('Recordaction.<FQCN>')    references the Java RecordAction class
```

`Agentactiontool` and `Agentactioncontext` sit at the same indent level as the `Agentaction` blocks inside
the parent `Agent(...)`, not nested in the action. Forgetting them imports cleanly and the action then
behaves as if it has no tools or context: the LLM says it does not have access to that information instead
of the MDL failing.

## 1. Aicontexttype (custom)

```sql
RECREATE Aicontexttype <api_name>__c (
    label('<Display Label>'),
    active(true),
    admin_configurable(true),
    runtime_code('Aicontexttyperuntimecode.<package>.<ClassName>')
);
```

`admin_configurable` is required, and `true` is what makes the context type configurable on a custom
agent. The component reference marks the attribute read-only, while the platform's own custom-context-type
walkthrough sets it in `RECREATE`, so treat a rejection on that clause as expected rather than a mistake in
the file. Do not add a `configuration_code(...)` clause; Vault stamps it and `configuration_layout()` as empty
parens on the round trip regardless `~`. Scope is not an MDL attribute; DOCUMENT, OBJECT, or ALL is
declared on the Java handler's annotation.

**Prefer a standard context type first.** Vault ships context types for the current document's metadata and
content, for the content of related documents, and object-scoped equivalents exposing the current record's
fields and rowsets of related records. They need no Java and bind from the Agent's MDL through a populated
`context_configuration(...)`. Only go custom when the data need is genuinely not covered. The names used
below (`document_data_content__sys`, `object_data__sys`, `related_document_data_content__sys`) match the
shipped labels, but confirm them against the target Vault `~` by listing `Aicontexttype` through the
configuration endpoint and filtering to `__sys`, which is the api skill's call.

> **Data "tagged on the document or record" is a relationship, not a reason to write Java.** Data living in
> another object (countries, products, related courses, regulatory regions) is reachable from the scope: a
> document object-reference field is a `documentFields` Rowset, an inbound child collection on the scoped
> record is an `objectFields` Rowset, and rowsets nest, so an object reference's own children come too. Ask
> first whether the data is tagged on or linked to the current document or record. If yes, standard context
> plus a Rowset. If it is a global list with no relationship, such as every country in the Vault regardless
> of what the document is tagged with, custom Java or a VQL Query tool.

**A custom context type cannot take per-binding parameters.** Every `Agentcontext` referencing one uses
empty `context_configuration()`; a populated value fails, since there is no configuration handler to route
it to `~`. The workarounds are on the Java side: a constant in the handler for a one-off, an abstract base
class plus a thin per-instance subclass each with its own `Aicontexttype`, or a small settings object the
handler queries at runtime for a value an admin is meant to manage.

## 2. Aitooltype plus Aitool (custom Java tool)

**Both files are required.** An `Aitooltype` alone registers the runtime handler but is invisible to the
LLM and cannot be attached to an agent action. The `Aitool` is what the agent wires up.

File 1, `Aitooltype.<api_name>.mdl`:

```sql
RECREATE Aitooltype <api_name>__c (
    label('<Display Label>'),
    active(true),
    admin_configurable(true),
    runtime_code('Aitooltyperuntimecode.<package>.<ClassName>')
);
```

`runtime_code` is required on `Aitooltype` (optional on `Aicontexttype`). `admin_configurable(true)` makes
the tool type configurable on a custom agent and can only be `true` on a custom tool type.
`support_custom_action` defaults to `true` and gates whether a tool built from this type can be assigned to
a custom agent action. Deployed handlers of either kind appear in the Vault UI under Admin, Configuration,
Tool Type Handlers and Context Type Handlers, the quickest confirmation that a handler registered at all `~`.

File 2, `Aitool.<api_name>.mdl` (static mode; omit `input(...)` for dynamic mode, where the schema comes
from `onGenerateInput` at runtime):

```sql
RECREATE Aitool <api_name>__c (
   label('<Display Label>'),
   active(true),
   configuration(),
   description('<LLM-facing: what the tool does AND when to call it, max 1024 chars>'),
   input('{
  "type": "object",
  "properties": {
    "<field>": {
      "type": "<string|number|integer|boolean|array|object>",
      "description": "<required, non-empty: the only hint the LLM gets about this field>"
    }
  },
  "required": ["<field>"]
}'),
   tool_type('Aitooltype.<api_name>__c')
);
```

- **`description` is required and capped at 1024 characters.** It reaches the LLM at tool-selection time,
  so it says what the tool does and when to call it. `configuration` and `input` cap at 32000 each. A tool
  that is not `active(true)` is invisible to the LLM at that same moment `~`.
- **Every schema node needs a `description`**, every `properties.<key>` and every `items`. A missing one
  fails the import.
- **Do not omit `input(...)` for a zero-parameter static tool.** Write
  `input('{"type":"object","properties":{},"required":[]}')`. The clause is formally optional only because
  dynamic-mode tools supply the schema at runtime.
- The `Aitool` API name need not match the `Aitooltype` API name, since one tool type can back several tool
  instances, but this skill reuses the same name unless told otherwise. What `RECREATE` does on install and
  redeploy is the mdl skill's.

## 3. Aitool wrapping a standard tool type (no Java)

Two standard tool types are documented: **VQL Query**, which runs a preconfigured VQL query, and **JSON
Response**, which generates JSON structured data from an input schema for an SDK or API caller. Wrapping
either is a single MDL file: no `Aitooltype`, no Java, no compile step, since the runtime handler is
already on the platform image.

```sql
RECREATE Aitool <api_name>__c (
   label('<Display Label>'),
   active(true),
   configuration({<vault:configuration xmlns:vault="VeevaVault">
     <vault:input type="string" name="vql_query__sys">SELECT id AS person_record_id, name__v FROM person__sys WHERE name__v LIKE '$\{name_fragment\}%' ORDER BY name__v</vault:input>
   </vault:configuration>}),
   description('<LLM-facing: what the tool does AND when to call it, max 1024 chars>'),
   input('{"type":"object","properties":{"name_fragment":{"type":"string","description":"..."}},"required":["name_fragment"]}'),
   tool_type('Aitooltype.vql_query__sys')
);
```

**Brace escaping is mandatory inside `configuration({...})`.** MDL parses the outer braces as the clause's
delimiter, so any literal `{`, `}`, or `${` meant to reach the runtime is escaped as `\{`, `\}`, `$\{`. A
runtime token `${name_fragment}` is typed `$\{name_fragment\}`, closing brace included. Forgetting this
either fails the import on a brace mismatch or, worse, imports clean and the literal text is never
substituted at runtime.

**Every `${param}` token in the query body needs a matching `properties.<param>` entry in `input(...)` and
the reverse**, no orphans either direction. Property names are case sensitive. Supported parameter types
are string, integer, and number. The `<vault:input>` element takes one slot per configuration, named for
the tool type.

The full limit list and the escaping rules live in Vault Help; pull them with the docs MCP (source
`vault_help_documentation`, query `Configuring Agent Tools`) rather than restating them from memory. The
four that force a rewrite rather than a tweak: no `LIMIT`, `PAGESIZE`, or `PAGEOFFSET`; object and document
targets only; ten input parameters at most; a hard 1,000-record result, past which only the first 1,000
reach the LLM. A `CONTAINS` with several arguments takes several individual tokens, and `FIND` takes its
search phrase as one token, so the AND or OR has to be inside the value the LLM supplies.

**Writing the query text itself is the vault plugin's query skill's job**; this one owns the
wrapper. Hand over the body and let that skill settle aliasing, `TONAME(...)` on label-storing fields,
`FIND` versus `LIKE`, case sensitivity, and relationship traversal. Three things are the wrapper's business
and worth stating in the handoff:

- **A parameter substitutes into a value position only** (`= ${x}`, `LIKE '${x}%'`), never a column name,
  function name, operator, or `FROM` target, since Vault parses those at import time, before
  substitution `~`.
- **Anything returning several rows needs an `ORDER BY`**, so the most relevant land inside the
  1,000-record cap, plus an `AS` alias on every function call and every `id` column. An unaliased function
  comes back as its literal expression text, and a bare `id` tells the LLM nothing about which object it
  belongs to. Convention: `id AS <object>_record_id`.
- **A tool returning records the agent should hyperlink is a two-file change** `~`. Link generation fills
  the record id from the tool's aliased id column, but the object API name comes from the calling agent's
  objective and its enumeration of in-scope objects, not from the tool. Records of an object the objective
  does not enumerate render as plain text, since the LLM will not invent the API name. Author the `Aitool`
  and add the object to the calling agent's link-generation block. Same asymmetry on subtyped objects: the
  type API name is what a filter matches on, the base object API name is what a deep-link path needs.

**For a JSON Response tool, do not write a description saying the tool outputs JSON.** Describe what it
reports instead; wording that says the tool produces a JSON result can error when the action invokes it.

Branching logic, multi-step orchestration, a result over 1,000 rows, or a target that is neither object nor
document does not fit a standard tool type. Fall back to a custom Java tool (section 2).

## 4. Agent (new agent definition)

One MDL file holding the top-level `Agent` block, its `Agentaction` children, and the peer wiring
subcomponents. Condensed shape, one of each:

```sql
RECREATE Agent <api_name>__c (
    label('<Label>'),
    active(true),
    agent_user('[agent]<api_name>__c'),
    class('Agentclass.com.veeva.vault.system.agent.<BaseDocumentAgent|BaseObjectAgent>'),
    description('<Admin-facing and chat-routing-facing description, max 500>'),
    document_type('Doctype.<dt>'),          /* DOCUMENT scope only; empty parens for OBJECT */
    object('Object.<obj>__c'),              /* OBJECT scope only; empty parens for DOCUMENT */
    model_level('<BASIC|ADVANCED>'),
    objective('<System-prompt-level role and guardrails: ground every statement in provided
context, never invent data, say "Unknown" for missing values>'),

    Agentaction <action>__c (
        label('<Action Label>'),
        active(true),
        api_access(false),                  /* true only if invoked over the Vault API or MCP */
        description('<what this action does, for chat routing and the LLM, max 500>'),
        instructions('<see the three-part contract below>'),
        supplemental_instructions(),
        support_chat(true),
        hide_in_chat(false),
        tool_evaluation('<AUTO|REQUIRED>'),
        tool_evaluation_required_tool()     /* names an Aitool when tool_evaluation is REQUIRED */
    ),

    Agentactiontool <action>_tool_<tool>__c (
        label('<wiring label>'),
        active(true),
        action('Agentaction.<action>__c'),
        agent('Agent.<api_name>__c'),
        tool('Aitool.<tool>__c')
    ),

    Agentcontext <ctx>__c (
        label('<context label>'),
        active(true),
        always_include(false),
        context_configuration(),            /* empty for a custom Aicontexttype; populated XML
                                                for a standard one, see below */
        context_type('Aicontexttype.<ctx>__c'),
        description('<what this context gives the LLM, max 500>')
    ),
    Agentactioncontext <action>_ctx_<ctx>__c (
        label('<wiring label>'),
        active(true),
        action('Agentaction.<action>__c'),
        agent('Agent.<api_name>__c'),
        context('Agentcontext.<ctx>__c')
    )
);
```

Requiredness and length caps for every attribute above are in the component reference (docs MCP, source
`vault_developer_documentation`, query `Agent MDL component type`). What that table will not tell you:

- **Three descriptions are LLM- or routing-facing, not admin notes.** Chat routing reads the Agent's to
  decide whether to hand a request over, the LLM reads the `Agentaction`'s to understand the action and the
  `Agentcontext`'s to use the context. All three cap at 500 characters, and an `Aitool`'s caps at 1024;
  `objective`, the system prompt, and `instructions` are the long-form fields.
- **Routing is driven by those descriptions.** Per the Agent MDL page, Vault uses the agent's
  `description` for chat routing, so it says what the agent is for in plain words. The action
  descriptions shape the match too, and a request that only partly matches gets a clarification
  question.
  Use the full budget and list every kind of question the action answers (a status breakdown,
  a filtered list, a count per related record), not just the headline use case. Try five or six off-center prompts before
  calling an agent done.
- **A question over a set needs one set-level tool, not a per-record tool called in a loop.** A
  per-record VQL tool applied to every member of a record set makes the model call it once per record, and
  that run is slow and ends in failure. Give the action a Java tool that walks the set and queries in
  batches, and keep the per-record tool for one named record.
- **`class(...)` has to match `document_type` / `object`.** Document base class with DOCUMENT scope, object
  base class with OBJECT scope, the other of the pair left as empty parens. A mismatch imports clean and
  the agent never resolves a scope source at runtime `~`. Read the base class names off an existing agent's
  retrieved MDL.
- **Component references carry a literal type prefix**, `Doctype.`, `Object.`, `Objecttype.`; a bare API
  name imports but never resolves `~`. `object_type` is optional and narrows an OBJECT-scope agent to one
  type.
- **`agent_user('[agent]<api_name>__c')`.** The literal `[agent]` prefix points Vault at the agent service
  user, which is what lets actions run with nobody at the keyboard. Never substitute a real user or drop
  the brackets `~`.
- **`api_access` controls visibility to the Vault API and the MCP server**, not one endpoint: with `false`,
  no API or MCP call to the action is possible. `hide_in_chat(true)` is rejected `~`; the valid forms are
  empty parens or `false`.
- **`tool_evaluation('REQUIRED')` needs a populated `tool_evaluation_required_tool(...)`** naming a tool
  also wired to the same action through an `Agentactiontool` peer. `AUTO` leaves it empty.
- **`always_include` and `Agentactioncontext` presence are inverse.** `always_include(true)` means no
  `Agentactioncontext` peer for that context, since it attaches to every action already.
  `always_include(false)` means at least one peer is required, or the context is wired to the agent,
  attached to no action, and silently invisible.
- **A tool whose input schema takes a scope id** (`document_id`, `record_id`, `version_id`, or any property
  described as the current document or record) **needs a context binding that surfaces that id.** The
  implicit scope on execution does not make the id visible to the LLM as a structured fact, and without a
  binding the model invents one. Bind a lean context carrying just the id rather than the full document or
  record payload when the action only relays it `~`.

> **Extending an agent that already exists on the Vault.** The agent usually lives only on the Vault, not
> in the repo. Retrieve its complete MDL (the api skill's call), save it verbatim, edit only what is
> changing, then redeploy through the mdl skill. `RECREATE` replaces the whole component, so any
> subcomponent dropped from the file is deleted from the live agent; never hand-author a partial agent
> block over an existing one. The same property makes temporary test scaffolding clean to remove: add an
> action, test, then redeploy without it.

Agents are Vault-specific and cannot reach data outside their own Vault, and a Vault ships standard agents,
so list what is already there before scaffolding a duplicate. That call, and resolving a doctype or object
named in UI language to its API name, go to the api skill; surface the matched API name for
confirmation before emitting.

### `context_configuration` for a standard context type

An `Agentcontext` wrapping a standard type needs the XML envelope populated, or the LLM sees no data from it.

Per-key rules for the document content type `~`. `documentContentTypes` takes `"TEXT_EXTRACT"`,
`"PAGE_IMAGES"`, and `"SOURCE_FILE"`, which combine and give extracted plain text, per-page renditions, and
the raw binary. `useStructuredText` is required whenever `documentContentTypes` is supplied, not a
free-defaulting optional flag, and omitting it fails the import; `true` preserves section and heading
structure instead of a flat paragraph stream, which is what any agent reasoning over headings wants.
`documentVersion` picks the version, usually `"SELECTED"`, the one the user is looking at. `documentFields`
groups metadata under named sections with `type` one of `JsonArray`, `JsonObject`, `Boolean`, or `String`;
an unrecognized `name` is dropped silently, so a misspelled key costs the whole value with no error.

**Placeholder convention: label as key, value as value.** `${<field_api>}` resolves to the current record's
value, type-cast and serialized, and belongs in the value position. `${<field_api>;label}` resolves to the
field's localized label and belongs in the key position. Emit `"${<field>;label}": "${<field>}"` so the LLM
sees self-describing pairs rather than an opaque API name.

The envelope:

```xml
context_configuration({<vault:configuration xmlns:vault="VeevaVault">
    <vault:input name="objectFields" type="JsonObject">\{
        "<Section Label>": \{
            "data_source": \{ "label": "<Section Label>", "description": "<what this is>" \},
            "data": [
                \{
                    "$\{field__c;label\}": "$\{field__c\}"
                \}
            ]
        \}
    \}</vault:input>
</vault:configuration>})
```

Same brace-escaping rule as section 3's `configuration({...})`: every literal `{`, `}`, and `${` inside the
JSON body is escaped. Missing escapes fail the import with a malformed-XMLString error naming
`context_configuration`.

**Two valid `objectFields` structures** `~`. A flat field map of label-to-value pairs deploys directly and
covers the current record's own fields. The `data_source` / `data` envelope above is what exposes related
or child records. Each named block has two parallel halves: `data_source` carries the descriptors (`label`,
`description`, an optional `extended_description`, and for a related collection the `source` holding the
Rowset expression), `data` an array with one template object, the field map for the block. Child
collections nest by repeating the block name in both halves, to arbitrary depth. `documentFields` uses the
identical envelope.

Rowset grammar, inside `data_source.source`:

```
${Rowset(<rel>__cr [where <vql_where>] [order by <ord_field>] [maxrows <n>])}
```

- `<rel>__cr` is the inbound relationship API name, the same one a VQL subquery against the parent object
  would use.
- `where <vql_where>` is an optional VQL predicate with no `WHERE` keyword, supporting `=`, `!=`,
  `IN (...)`, `AND`, and `OR`.
- `order by <ord_field>` is optional, with `ASC` or `DESC` appendable.
- `maxrows <n>` is optional and otherwise falls back to a Vault-wide limit `~`. Cap it explicitly on a
  large set.

Placement and source rules that fail the import when broken `~`:

- **A Rowset expression is valid only as the value of `data_source.source`.** As a JSON key, or as a bare
  label-keyed string value, it is rejected.
- **For `documentFields`, the Rowset source is the document relationship name, not the field name.** Use
  the `relationshipName` from the doctype metadata; the field name fails with an error saying the name does
  not map to a valid record or document field. A single-value reference still works, yielding zero or one
  rows.
- **Multi-hop is nested rowsets, never a chained `a.b` inside one Rowset**, which is rejected as an invalid
  object relationship. Nest a second `data_source` / `data` block whose inner Rowset source is a
  relationship on the related record's object. Inside a rowset's `data` map the fields are the related
  record's fields, outbound lookups included.

**Static literal values.** An entry whose value contains no placeholder token passes through to the LLM
verbatim as a constant, which lets a small dedicated context inject fixed reference data such as a URL base
or a formatting hint that the action's `instructions(...)` then interpolates. Keep it separate from the
record and Rowset context, often `always_include(true)`. Static values are still subject to brace escaping.

### `instructions(...)` contract

Curly-brace tokens in `instructions` are read as field references, so a placeholder such as
`{name}` for a tool's output key gets the Agent rejected with `GEN-FREC-II-1284` ("instructions must
reference valid Object or Document fields"). Describe the tool's output in prose, and use a brace
token only for a real field of the agent's object or doctype.

An instruction that asks for a count or a total in the answer needs a tool that returns that number, such
as a VQL aggregate (`SELECT COUNT(id), SUM(amount__c) FROM expense__c WHERE status__c = 'unsubmitted__c'`)
or a handler that computes it. Otherwise the model adds up the rows itself, and it can
be wrong on a table it listed correctly. If no tool returns the figure, leave the count or total out of the
instructions and let the table speak for itself.

Every `Agentaction.instructions(...)` body carries three parts, in order: a sentence naming every bound
context and tool and what each provides; an explicit Markdown output template with named placeholders; and
a negative-rule list covering no preamble, no closing remarks beyond a short follow-up, Markdown only, and
explicitly no raw HTML. Write the no-HTML rule in plain English, naming anchor, button, image, div, and
span tags in words: literal markup inside the instructions body reads as the thing being demonstrated
rather than the thing being forbidden `~`.

What the instructions can promise is bounded by how the output renders `~`. Agent text is processed as
CommonMark, so links, emphasis, inline code, fenced blocks, lists, and headings all work, while raw HTML
does not render at all and the user sees the angle brackets in the bubble. Only `href` survives on a
rendered link, so there is no agent-side way to make a chat link open in a new tab; say so rather than
asking the LLM for a behavior it cannot produce. A same-host hash URL such as `https://{vault_dns}/ui/#<route>`
navigates through the Vault UI with the chat panel staying open, though the click appears to do nothing
when the user is already on that route. Default link styling is solid blue with no underline, which some
users do not read as a link, so a leading icon character inside the Markdown label makes clickability
obvious.
