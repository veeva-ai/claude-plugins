---
name: ai-metadata
version: "0.1.0"
description: >
  For a Vault admin, or the SME who knows what the data means, who needs Vault AI metadata where no
  delivered metadata covers the data model: which custom objects, object types, document types and
  picklists, and which standard objects with no delivered entity, should get it, and what the
  descriptions, synonyms, fields, relationships, metrics and values should say. Use it for "draft AI
  metadata for our custom objects", "which objects need Vault AI metadata", "the agent keeps
  confusing these two objects", "add synonyms for AI Chat", "add a metric for how long these take",
  "how many AI metadata entities do we have left", or when Vault AI Chat answers from the wrong
  object. It reads a pulled workspace, drafts Vsm files and a review file for the SME, and checks the
  draft offline. Not for running the change (vault-administrator → mdl), pulling (config), record
  counts (vault → query), agents (vault-developer → ai-agents), sign-in (vault → login), or the
  search thesaurus.
---

# Vault AI Metadata

Agents in Vault AI Chat see little more than names and labels for anything no AI metadata
describes, so a custom object with a vague label, or one that looks like a delivered object, gets
picked wrongly or not at all. This skill reads a pulled workspace, works out which targets need
metadata and in what order, drafts it as Vsm component files in the workspace layout with a review
file for the SME, checks the draft against the workspace, and hands the files to the
vault-administrator plugin's mdl skill to run. It never writes to a Vault.

It works from the attributes the platform defines for Vault AI metadata and builds in nothing
app-specific. For an app's standard objects it reads what the Vault itself holds.

## Scope

- Reading the AI metadata a Vault already has, delivered and custom, from a pulled workspace.
- Choosing and ranking the targets that need metadata, against the per-Vault limits.
- Drafting entities, fields, relationships, metrics, picklists and values as RECREATE files.
- Checking a draft, its own or one the user brings, before it goes to mdl.
- The review file for the SME, applying the edits it comes back with, and the question bank for
  the before-and-after check in Chat.

Out of bounds:

- Executing a drafted change, and its dry run. That is mdl's job, and it keeps its plan stop.
- Pulling or refreshing the workspace. That is the config skill.
- Counting records or documents, or reading field values. That is the vault plugin's query skill.
- Building or changing an agent, its actions or tools, or the agent context that uses the metadata.
- Turning on Vault AI or granting the AI Metadata permissions, which is Admin UI work.
- The search thesaurus in Admin > Settings > Search Settings. It widens search results and is not
  agent metadata.
- Editing delivered (standard) AI metadata, which Vault does not allow.
- Moving AI metadata to another Vault. That is a package, through the vpk skill.
- Signing in, and any file that holds a username, password or token.

## Handoffs

| Need | Goes to |
| --- | --- |
| Run the drafted files, with a dry run first | vault-administrator → mdl |
| A workspace, or a fresher one | vault-administrator → config |
| Record and document counts, field values for evidence | vault → query |
| Agent instructions, context, actions and tools | vault-developer → ai-agents |
| Moving metadata between Vaults | vault-administrator → vpk |
| Sign-in and credentials | vault → login |
| Enabling Vault AI, AI Metadata permissions, search synonyms | Vault Admin UI |

Stop at the boundary and name the owner. When the user asks this skill to push, deploy, skip the
dry run or set up a login file, the answer is the hand-off, whatever the wording.

## Step 1: Read what the Vault already has

Answer from a pulled workspace and quote the paths. The Vault's AI metadata is under
`components/Vsmentity/` and `components/Vsmpicklist/`; the data model it describes is under
`components/Object/`, `Objecttype/`, `Doctype/`, `Picklist/` and `Objectlifecycle/`. When there is
no workspace, or nobody knows how current it is, the pull and its freshness check go to the config
skill, and the answer says it is only as current as that pull.

AI metadata comes down as two top-level types. A `Vsmentity` holds `Vsmfield`, `Vsmrelationship`
and `Vsmmetric` subcomponents in nested folders
(`components/Vsmentity/<entity>/Vsmfield/<field>/<field>.mdl`), and a `Vsmpicklist` holds
`Vsmvalue`. The attributes and their lengths are in `.vault/metadata/Vsmentity.json` and
`Vsmpicklist.json`. Read what an entity describes from its `target`, never from its name, because
the two differ on delivered components: `person__v` targets `Object.person__sys`.

When the workspace has no `Vsmentity` or `Vsmpicklist` folder, do not say the Vault has no AI
metadata. The type can be excluded in `.vdxignore`, the pull can be older than the metadata, or
Vault AI can be off on that Vault. Read `.vdxignore`, say which of the three it rules out, and
offer the pull. Which of the three a missing folder means is not yet confirmed, so say that too.

Delivered metadata cannot be edited, and an entity can only target an object, object type or
document type that no other entity uses. Treat every delivered `Vsmentity` and `Vsmpicklist`, and
everything under them, as read-only, and do not draft a change to one or a custom entity on a
target a delivered entity already holds. When a delivered description no longer fits how this
Vault uses the object, say so and name the agent's instructions and context, through the
vault-developer plugin's ai-agents skill, as where that correction goes.

## Step 2: Find the candidates

A candidate is active and not targeted by any entity, delivered or custom. That covers custom
objects, object types and document types (a custom subtype under a standard document type counts),
and also a standard object that no delivered entity holds, since an entity needs only an active
target no other entity uses. Whether Vault accepts a custom entity on a standard target is not
yet confirmed, so say that when you propose one.

Tabs are the strongest sign users will ask about something. List every object a `Tab` or
`Subtab` points at (`object('<name>')` in the files under `components/Tab/`) that no entity
targets. A standard object found this way goes on the list only when no delivered entity targets
it, with the custom fields it carries. When the workspace has no object file for it, say the pull
has no definition and do not draft it.

For an app's standard objects, work only from what the Vault holds: delivered Vsm components in
the workspace, and the object's and its fields' descriptions and help text. Do not fill in what an
app object means from general knowledge of the app.

Leave out join objects with no business meaning of their own, integration staging, reference and
configuration data, test and demo objects, and objects with no tab, no lifecycle and no records.
Say what was left out and why. `references/drafting-rules.md` has the ranking signals and the
exclusions in full.

## Step 3: Rank them and budget the slots

Rank on two questions: will users ask about it (traffic, a tab, a report or dashboard, a
lifecycle, a workflow), and will the agent misread it without help (a label that does not say what
it is, a look-alike object or document type, two reference fields an agent cannot tell apart).
Each proposed entity carries a one-line reason with both halves.

Reports and dashboards show what users look at. A report is on an object when its
`report_type('Reporttype.<name>')` names a report type whose `primary_object` is that object, and
a dashboard counts when its `dashboard_markup.xml` charts such a report (`report="Report.<name>"`).
A report says someone built a view of the object, not how often it is opened; the pull carries no
run counts. When the workspace has no `Report` or `Dashboard` folder, do not read that as a Vault
without reports, since `.vdxignore` can leave a type out; say the signal is missing from the pull.

Record and document counts are data. Get them through the vault plugin's query skill or ask the
user, and never infer volume from the files; without counts, say the ranking rests on structure
alone.

State the plan against Help's limits: 50 custom entities and 100 picklists per Vault; per entity 50
fields, 30 object relationships and 20 metrics; 50 values per picklist. Count the custom entities
and picklists the workspace already holds. Do not say whether delivered entities and picklists
count toward the two per-Vault caps. Propose
well under 50, and say the held-back slots are for what real questions turn up after launch.

Default to one entity on an object, with field metadata on its object type field. Propose object
type entities only when the types are different business things, since each one is another entity
against the 50. Whether an object and its own object types can each hold an entity at once is not
confirmed.

An agent reaches entities through its agent user's permissions, so metadata on an object that user
cannot read does not reach the agent. Name the agent user's profile as the thing to check.

If you picked the candidates yourself, stop once after the ranked plan and ask only which targets
to draft: confirm, drop or add, which of two look-alike targets is current, whether a target is
legacy or test content, whether the agent user can read it, and whether to hold some of the 50
back. Draft nothing until the user answers. Do not ask at this stop what a target or a field means
or which words users use; those questions go in the review file, beside their rows. If the user
named the targets, that is the confirmation, so draft.

## Step 4: Write descriptions and synonyms

Each description says how an agent should read the target: what it is for, what it is used to
find, filter or sum, when to use it and when not to. Name the look-alike it is not, and the right
one to use instead, in a clause that starts `IMPORTANT:`, the form delivered metadata uses. Never
restate the label, and never copy the target's help text word for word; help text is written for
the person entering data, not for an agent.

Put the nouns users type, and the ones that set the target apart from its look-alikes, in the
first sentence. A word that lives only in the synonyms can go unseen when Chat searches the
schema.

When a custom target stands in for a delivered entity (the same business concept under a custom
name), start from the delivered description, name the delivered entity it came from, and add only
what differs. A synonym copied the same way collides with that entity and goes through the
collision check below.

Synonyms are capped at ten per item. Help asks for jargon, abbreviations and terms an agent cannot
map on its own, not plurals or the label with "record" or "item" added. A synonym is the business's
own word only when it came from what the user supplied: a glossary, SOP titles, a search thesaurus
export, questions users have asked, or an SME. Every other synonym is marked proposed and listed
for the SME.

Check every drafted label and synonym against the other entities in the draft and against the
delivered entities in the workspace, and flag each collision. A collision with delivered metadata
is the serious one, since that text cannot be edited; the delivered Person entity, for one, carries
"personnel" and "contact".

Leave out a synonym that a standard object's label already holds once singular and plural are
folded, whether the label is in the object's file or in a delivered entity's `target_label`, and
list the word in the review file with that label. In Chat, a word that matches an object's label
can decide which object is read before any synonym does.

Then check the fields and relationships on each target for names a user could confuse: names that
differ only by a trailing number (`features__c`, `features1__c`), the same label twice, labels that
match once singular and plural are folded, one label that starts another, and a reference whose
label names a different object than the one it targets. Each pair gets an `IMPORTANT:` clause in
both descriptions saying which to use when. Numbered slots (`presenter_1__c`, `presenter_2__c`)
stay separate and are told apart, never merged. A mislabeled reference is described by what it
targets. The patterns are in `references/drafting-rules.md`.

## Step 5: Choose fields, picklists, relationships and metrics

Pick the fields users filter, group and ask about: status, key dates, owner, human-facing
identifiers, classification picklists, amounts. Leave audit and system fields out unless a
question needs one, such as a submitted date read from `created_date__v`. Hold a top entity to
about 10 to 20 fields; more fields can pull questions to the wrong entity. A field goes on one
entity at most, since two entities cannot reference the same field.

Before choosing, get each candidate field's fill rate and a few sample values through the vault
plugin's query skill, or ask the user for them; without them, say the choice rests on the
configuration alone. Leave out a field filled on few records unless the user or a question names
it, and give its fill rate in the review file. What a field holds comes from those values. When
they don't show what a field means, ask the SME; do not guess. Example values go only on Text
fields, up to ten, and only where users type recognisable terms.

Rank the fields in this order: list columns (`list_column(true)` in the pulled field file) and
the fields in the first section of the object's page layout that holds fields, in the order they
appear there; then
fields a report or dashboard selects, filters or groups on; then fields in later sections, with a
system section last. The layout is `page_markup.xml` under `components/Pagelayout/`, found by its
`controller` (`Object.<name>` or `Objecttype.<object>.<type>`). A report's fields are the
`<object>.<field>` properties in its `definition.xml`, and a dashboard's groupings are in its
`dashboard_markup.xml`. The fill rate outranks the position: a mostly blank field in the first
section stays out, and a list column filled on every record stays in. Note why each field was
picked as you go, since the review file shows it.

Before a description names a picklist value or a lifecycle state, open the pulled `Picklist` or
`Objectlifecycle` files and use the labels there. Never name a value or state those files do not
hold.

A field whose picklist has its own `Vsmpicklist` points at it with
`vsm_picklist('Vsmpicklist.<name>')`, not at the Picklist. Help allows a field-level override of a
reused picklist's description and synonyms, and the pulled field files carry `vsm_picklist_local`
for it. Do not draft an override until that shape is confirmed.

Relationships are for object entities only, and only the entity's own reference fields. `target`
is the relationship name (`<name>__cr`), `target_object` the bare object name, and the component is
named for the relationship with `__c` in place of `__cr` (`manager__c` targets `manager__cr`). Add
one only where the traversal is ambiguous, two reference fields to the same object first (the
employee on a request and the manager who approves it). Carry `cardinality('MANY_TO_ONE')` and an
empty `join_object()` on a reference. Whether they can be left out, and
how a many-to-many spells its `join_object`, is not confirmed; say so when you draft one.

Draft a metric only when a question combines fields, such as a duration between two dates or an
aggregate across objects, and never as a single field under another name. `target_fields` names
fields on the entity's own object, and `instructions` say how to calculate it, from which fields
and under what conditions, without restating the description. When a metric needs more than ten
target fields, say that more than ten is not yet confirmed, and state no limit. Say whether you
wrote the instructions as a plain sentence, the form every delivered metric in a pull uses, or as a
Vault formula or rule, the form Help asks for.

A `Vsmpicklist` targets `Picklist.<name>` or `Objectlifecycle.<name>`, and a `Vsmvalue` the bare
value or state name. Document lifecycles can carry picklist metadata too, but the target form for
one is not yet confirmed, so say that before drafting one. Describe a value only where its
label does not say what it means, and a lifecycle state where words like "open" or "approved" mean
something specific in this process.

## Step 6: Write the files

Every string fits the pulled type metadata: an entity description up to 1,000 characters; field,
relationship, metric, picklist and value descriptions up to 500; metric instructions up to 2,000;
labels up to 60.

Every draft is RECREATE files in the pull's layout: a folder per component, subcomponents in nested
folders referenced as `Vsmfield(@Vsmfield/<name>.mdl)`. Start each one from a pulled Vsm component
of the same type, copy it, rename it and change only what the request needs; new names go in the
`__c` namespace. The shapes come from pulled files, and `references/vsm-shapes.md` holds an
example of each type. Never write a shape from memory. RECREATE
creates a component that does not exist yet, so a new entity is RECREATE too.

Every subcomponent name is `__c`, even when its target is a standard field: `record_name__c`
targets `name__v`. A subcomponent named `name__v` fails the push with "[standard] sub-components
cannot be created" (GEN-FADDS-ONA-1019). Standard fields still get metadata this way, so never
leave one out only because it is standard.

Write an apostrophe inside a string as two single quotes (`employee''s`).

When the workspace already holds a custom entity for the target, start from that entity's files and
keep what it has, because a RECREATE replaces the whole component and anything missing from the
files is gone after the push. Keep an existing metric's `target_fields` as they are; a metric that
needs other fields gets a new name. That Vault rejects a change to them on RECREATE
(GEN-FREC-II-1061) is not yet confirmed, so say so when it matters.

## Step 7: Check the draft

Before any hand-off, check the draft against the workspace and list every problem, whether the
draft is yours or the user's:

- a string over its pulled length, or more than 50 fields, 30 relationships, 20 metrics or 50
  values
- an empty description (the type metadata marks it required)
- an apostrophe not written as two single quotes
- example values on a field that is not Text
- a `vsm_picklist` that names a Picklist instead of a `Vsmpicklist`
- a subcomponent name outside `__c`
- a `target` that names a field or relationship the pulled object does not have, or one that is
  inactive
- a label or synonym that collides with a delivered entity or another drafted one, or a synonym
  that a standard object's label already holds

An entity description up to 1,000 characters passes. Offer the
fixed files, keeping the user's wording wherever it is not the problem. The check list with the
source of each line is in `references/drafting-rules.md`.

## Step 8: Write the review file and the question bank

Write the review file beside the workspace, not inside it, so a pull or push never picks it up.
It is markdown in this release, written for the SME to edit in place and hand back: one table,
with one row per drafted entity and one per drafted field, relationship, metric, picklist and
value. A subcomponent's Target is written `<entity> . <name>`. The columns, in this order:

| Column | Who fills it |
| --- | --- |
| Target (read only) | you |
| Description (edit here directly) | you; the reviewer edits it in place |
| Synonyms (edit here directly) | you; the reviewer edits it in place |
| Basis (read only) | you, when you draft the row |
| Claude's question (read only) | you, only where the row needs an answer |
| Reviewer answer / comment (write here) | the reviewer |
| Reviewer (admin assigns) | left blank for the admin |

Record the Basis when you draft the row, never afterwards. It is `general knowledge`, or the
Vault source the text came from, named exactly: a label, help text, description or lifecycle state
from the configuration, a delivered entity, or a query result. Never name a source you did not
read. Synonyms that did not come from the user are marked proposed there. A field row also says
why the field was picked: list column, layout section, report or dashboard, fill rate, sample
values. A confusable pair shows in both rows' descriptions, with the evidence in Basis.

Open with a short summary: the count against the 50 custom entities, the confirmed plan, and what
was assessed and left out, with the reason. Then the key rows: every `general knowledge` row, and
the top-ranked entities with their fields. Put the remaining rows in a second section marked
optional, so a reviewer can stop after the key rows. A question sits in its own row; never gather
questions in a list of their own.

Leave a question bank: five to ten questions per top entity in the business's own words, each term
checked against the drafted labels and synonyms, as the before-and-after check in Vault AI Chat. A
successful push does not show the agent answers better. Vault synchronizes active components to
the agents' vector store, and the question bank is the check.

Say too that AI metadata does not decide whether Chat searches documents or records; Chat can make
that choice from the words in the question before it searches the schema. So a question that needs records names the object the way users say it, and when
Chat goes to documents instead, a synonym or a field description is not the fix.

## Step 9: Apply the review, then hand off to mdl

When the reviewer hands the file back, diff each Description and Synonyms cell against the
component files you drafted, read each answer and comment, and write the changes into those files:
edited text as the reviewer wrote it, with apostrophes as two single quotes; an answer worked into
the description it concerns; a dropped row's file removed along with its reference in the entity.
Rerun the Step 7 check, collision lines included. A synonym the reviewer adds that collides with a
delivered entity, or that a standard object's label already holds, stays out of the files and goes
back as an open item. Then write a fresh review file, in the same columns, with only what is still
open, and say which rows changed and how. Repeat for as many rounds as the reviewer wants, and hand
off after the last one.

Execution goes to the vault-administrator plugin's mdl skill, which runs its dry run and its plan
stop against the Vault it is signed in to. Do not write to a Vault, run `vault config push` or
`vault mdl`, or offer to skip the dry run. Say that Vault AI has to be enabled on the target and
that whoever runs it needs the Admin: Configuration: AI Metadata Create and Edit permissions.

Say too that an entity's target cannot be changed after it is saved, so a wrong target means a drop
and a new entity, and that dropping an entity deletes its fields, relationships and metrics and
takes it out of any agent context that uses it. Before offering a drop, search `components/Agent/`
for the entity's name, and say that how agent context names an entity is not yet confirmed.

## Step 10: Requests to turn down

- "Change the description on Person" or any delivered entity: say delivered metadata cannot be
  edited and a custom entity cannot take its target either, draft nothing for it, and point the
  correction at the agent's instructions and context through ai-agents. If the business concept
  lives in a custom object, offer an entity on that.
- "Push it tonight", "no dry run", `--yes`, `--execute`, `--replace`: hand the files to mdl and say
  it keeps its dry run and plan stop. A RECREATE replaces everything the target entity holds, so
  overwriting one in production starts with reading it.
- A login file with a username and password, or a source and target Vault in one config: write no
  credential anywhere; sign-in is the login skill, and moving metadata between Vaults is a package
  through vpk.
- "How many entities do we have left" from a workspace with no Vsm folders: give the limits, no
  remaining number, and offer the pull, or a read-only count of `Vsmentity` rows in
  `vault_component__v` through query once a Vault is reachable.
