# Drafting rules

Sources: Help (Configuring Custom Vault AI Metadata, page 1119332), pull (what a pulled workspace and its type metadata hold), push (what Vault accepted or rejected when drafted files were pushed), practice (rules carried over from earlier AI metadata tools), and SME review (an SME's review of a draft). Each section says which it rests on.

## Choosing candidates (Help, practice, pull)

A target qualifies when it is active and no entity, delivered or custom, targets it (Help). Look for candidates in this order:

| Signal | Where it is in the workspace |
| --- | --- |
| A tab or subtab points at it | `object('<name>')` in `components/Tab/<tab>/<tab>.mdl` or `components/Tab/<tab>/Subtab/<sub>/<sub>.mdl` |
| A report or dashboard is built on it | `report_type('Reporttype.<type>')` in `components/Report/<report>/<report>.mdl`, that type's `primary_object('<name>')` in `components/Reporttype/<type>/<type>.mdl`, and `report="Report.<report>"` in `components/Dashboard/<dashboard>/dashboard_markup.xml` |
| It has a lifecycle or a workflow | `components/Objectlifecycle/`, `components/Objectworkflow/` |
| Its label does not say what it is, or it looks like another object | the object file and the delivered entities' targets |

Leave out, and say why: join objects with no business meaning of their own, integration staging, reference and configuration data, test and demo objects, and objects with no tab, no lifecycle and no records.

A standard object qualifies only when no delivered entity targets it. List the custom fields it carries (`Field` files ending `__c` under its object folder), since those are what users added to it. A tab can point at a standard object the pull has no file for; say so and leave it undrafted.

## Ranking (Help, practice)

| Question | Raises the rank |
| --- | --- |
| Will users ask about it? | traffic (record counts from query, never guessed), a tab, a report or dashboard on it, a lifecycle, a workflow |
| Will the agent misread it? | a label that does not say what it is, a look-alike object or document type, two reference fields an agent cannot tell apart |

Each proposed entity gets one line with both halves. Without counts, say the ranking rests on structure alone. A report or dashboard shows that someone built a view, not how often it is opened (pull). A missing `Report` or `Dashboard` folder can be a `.vdxignore` line, not a Vault without reports.

## Choosing fields (SME review, practice, pull, Help)

Get each candidate field's fill rate and a few sample values through query before choosing. Leave out a field filled on few records unless the user or a question names it, and give the fill rate in the review file. Hold a top entity to about 10 to 20 fields. A field goes on one entity at most (Help).

| Rank | Where it is in the workspace |
| --- | --- |
| 1. A list column, or in the first section of the page layout that holds fields, in layout order | `list_column(true)` in the field file; `<vault:field reference="<field>"/>` in the first `<vault:section>` with fields in `components/Pagelayout/<layout>/page_markup.xml` (a layout can open with a page or section that holds only a timeline or widget), the layout found by `controller="Object.<name>"` or `controller="Objecttype.<object>.<type>"` |
| 2. Selected, filtered or grouped on by a report or dashboard | `property="<object>.<field>"` in `components/Report/<report>/definition.xml`; `key="<object>.<field>"` in a dashboard's `dashboard_markup.xml` |
| 3. In a later layout section | the later `<vault:section>` elements |
| 4. Only in a system section | a section such as System or System Details holding the audit fields |

The fill rate outranks the rank: a mostly blank field in the first section stays out, and a list column filled on every record stays in. Before a description names a picklist value or lifecycle state, read it from `components/Picklist/<name>/Picklistentry/` or `components/Objectlifecycle/<name>/Objectlifecyclestate/`.

## Confusable names (practice, Help)

Check the target's own fields and relationships, and the entities in the draft, for these:

| Pattern | Example | What to write |
| --- | --- | --- |
| Names differ only by a trailing number | `features__c`, `features1__c` | an `IMPORTANT:` clause in both saying which holds what; ask the SME whether one is a leftover duplicate |
| Numbered slots | `presenter_1__c`, `presenter_2__c` | keep both, tell them apart; never merge |
| Same label twice | two fields labeled Features | `IMPORTANT:` clauses naming each other |
| Labels match after singular and plural folding | Release Theme, Release Themes | `IMPORTANT:` clauses |
| One label starts another | Application, Application Family | `IMPORTANT:` clauses |
| A reference labeled for one object but targeting another | labeled Application Family, targets `application__c` | describe it by its target and say the label suggests the other |
| Two reference fields to the same object | requester and approver, both Employee | a relationship on each, saying which to use for which question |

Delivered metadata writes these as a clause starting `IMPORTANT:` inside the description (pull).

## Words (Help, practice)

- Say what the target is used to find, filter or sum, and when not to use it.
- Never restate the label, and never copy help text word for word.
- Synonyms: up to ten per item; jargon, abbreviations and the words users type, not plurals or the label plus "record".
- A synonym is sourced only when it came from the user: a glossary, SOP titles, a search thesaurus export, questions users asked, or an SME. Every other synonym is marked proposed.
- Check every label and synonym against the draft and the delivered entities; a clash with delivered text is the serious one.
- Leave out a synonym that a standard object's label already holds, after singular and plural folding, and list it with that label.
- Put the nouns users type, and the ones that set the target apart, in the first sentence.
- AI metadata does not decide whether Chat searches documents or records; Chat can make that choice from the question's words before it searches the schema.
- A field's meaning comes from its values, read through query; when they don't settle it, ask.

## Check list before the hand-off

| Check | Source |
| --- | --- |
| Entity description at most 1,000 characters; every other description at most 500; instructions 2,000; labels 60 | pull (type metadata) |
| At most 50 fields, 30 relationships, 20 metrics per entity; 50 values per picklist | Help |
| No empty description; a metric has instructions and target fields | pull (type metadata marks them required) |
| Apostrophes written as two single quotes | push |
| Example values only on Text fields, at most ten | Help; GEN-FREC-II-1311 on other types is not yet confirmed |
| `vsm_picklist` names a `Vsmpicklist`, not a Picklist | pull |
| Every subcomponent name ends `__c`, whatever its target | push (GEN-FADDS-ONA-1019) |
| Every `target` exists on the pulled object and is active | Help (active targets) |
| An existing metric's `target_fields` unchanged | GEN-FREC-II-1061 on a change is not yet confirmed |
| No custom entity on a target a delivered entity holds; no change to a delivered component | Help |
| No label or synonym that collides with a delivered entity or another drafted one; no synonym a standard object's label holds | Help; SME review |
| A field on one entity at most | Help (multiple entities cannot reference the same field) |

## Review file (SME review)

Markdown, beside the workspace, never inside it. In this order:

1. Summary: the pull the draft came from and that nothing was pushed; the count against the 50 custom entities (limits, custom entities and picklists already held, this draft; whether delivered components count is not yet confirmed); the confirmed plan; what was assessed and left out, with the reason.
2. Key rows: every `general knowledge` row, and the top-ranked entities with their fields.
3. The remaining rows, in a section marked optional.
4. The question bank: five to ten questions per top entity in the business's words, each term checked against the drafted labels and synonyms.

One table for the rows, one row per drafted entity and per drafted field, relationship, metric, picklist and value:

| Target (read only) | Description (edit here directly) | Synonyms (edit here directly) | Basis (read only) | Claude's question (read only) | Reviewer answer / comment (write here) | Reviewer (admin assigns) |
| --- | --- | --- | --- | --- | --- | --- |
| employee__c | A person the company employs ... IMPORTANT: this is not Person ... | staff; team member (proposed) | general knowledge | Do contractors get Employee records? | | |
| employee__c . hire_date__c | The date the employee started with the company ... | date of hire (proposed) | config: label of `hire_date__c`; picked: first layout section, filled on 75 of 75 | Does it reset when someone is rehired? | | |

Basis is written when the row is drafted: `general knowledge`, or the source named exactly (a label, help text, description or lifecycle state from the configuration; a delivered entity; a query result). A field row adds why it was picked. A question goes only on a row that needs an answer; there is no separate question list.

When the file comes back: diff each Description and Synonyms cell against the drafted files, apply the edits and answers, remove dropped rows, rerun the check list, hold out any reviewer synonym that collides with a delivered entity or a standard object's label as an open item, and write a fresh file with only what is still open. As many rounds as the reviewer wants.
