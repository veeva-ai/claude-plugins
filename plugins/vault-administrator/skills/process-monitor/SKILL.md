---
name: process-monitor
version: "0.1.0"
description: >
  For a process owner or admin who wants a Process Monitor view built, explained or copied: "how
  long do complaints take from New to Closed", "cycle time by severity", "a process view for my
  lifecycle", "what does this process view measure", "copy this process view for 2025", "add a
  chart to my process view", first-time-right, rework or variant counts on a lifecycle. A Process
  View is a `Processview` component whose substance is a `settings` XML that Vault validates
  strictly, so this skill drafts it from pulled files and verified rules for the mdl skill to run. Not for a Vault report or dashboard (reporting), a one-off count (vault
  → query), or reading the results of a view that has run.
---

# Process Monitor Views

A Process View measures how records or documents move through a lifecycle: the time from a start
state to an end state, how many get there, how many repeat a state, and how that breaks down by a
field. Each view is a `Processview` component with a `settings` XML body and `Chart`
subcomponents, so a default `vault config pull` brings it down as files. This skill reads those
files, explains them, and drafts new views and clones as MDL. It does not write to a Vault, and
nothing runs a Process View except Process Monitor itself. The mdl skill executes what this skill
drafts.

## Scope

- Explaining what an existing Process View measures, from its pulled files.
- Drafting a new view on an object or document lifecycle, from a template or the nearest pulled
  view.
- Drafting a clone under a new name, with a checklist of what changed.
- Adding, changing or removing a chart, a filter, a metric, the timeframe or the target.

Out of bounds:

- Running a view, scheduling it, or reading its results. That is Process Monitor in Vault. No
  REST endpoint or CLI command runs one.
- Standard Process Views owned by Veeva, and the metrics reserved for them.
- Executing any change. Drafts go to the mdl skill.
- Vault reports and dashboards. Those belong to the reporting skill, even when they show cycle
  time.

## Step 1: Find the files

Process Views travel through `vault config pull`, mdl and push. If there is no workspace, hand
the pull to the config skill, then come back; do not guess state or field names in the meantime.
When the user only cares about Process Views, offer a `.vdxignore` that keeps just what this skill
reads; it makes the pull and every preview fast and leaves the other component types out of the
push plan. To explain or clone views, `Processview` is enough:

```
components/*
!components/Processview/
```

A new view also needs the states, fields, types and picklist entries Step 3 reads, so keep those
too. For an object lifecycle:

```
components/*
!components/Processview/
!components/Objectlifecycle/
!components/Object/
!components/Objecttype/
!components/Picklist/
```

For a document lifecycle, keep `Doclifecycle/`, `Doctype/`, `Docfield/` and `Picklist/` instead of
the object folders.

A pulled view sits here:

```
components/Processview/<name>/<name>.mdl
components/Processview/<name>/settings.xml
components/Processview/<name>/Chart/<chart>/<chart>.mdl
```

The parent `.mdl` names each chart as `Chart(@Chart/<chart>.mdl)`, one folder shallower than the
file. Keep that form exactly as pulled. To list the views a Vault has without a workspace,
`vault_component__v` filtered to `component_type__v = 'Processview'` returns them.

## Step 2: Explain a view

Read the view's `.mdl`, then `settings.xml`, then each chart. The meaning of every element is in
`references/settings-rules.md`. In the answer:

1. Name the process and the subprocesses (object types, or document types). On a document
   lifecycle, say that Process Monitor measures each steady-state version of a document
   separately, so one document can count more than once.
2. Give the cycle. The start step and how it is taken (first time in, or last time in), then the
   end step and how it is taken. Then any steps whose time is excluded, or that are the only time
   counted (`cycle_time_step_calculation`).
3. State the timeframe in plain words, naming the date it applies to.
4. State each filter in plain words, one line each. Read a picklist value back to its label from
   the workspace.
5. List the metrics, and the target with its unit when there is one.
6. List the charts: what each shows and what it groups by.
7. Give the owners, editors and viewers.

Quote the file paths so the user can open them.

## Step 3: Gather what a new view needs

Ask only for what the workspace cannot answer: which lifecycle, the start and end states, the
timeframe, filters, a target, and any charts. Everything else comes from the files.

- **States.** The bare state names are under
  `components/Objectlifecycle/<lifecycle>/Objectlifecyclestate/` or
  `components/Doclifecycle/<lifecycle>/Doclifecyclestate/`. Match the user's words to a state's
  label there.
- **Subprocesses.** For an object, the types are under `components/Objecttype/<object>.<type>/`.
  For documents, they are under `components/Doctype/`.
- **Fields.** For an object field, read `components/Object/<object>/Field/<field>/<field>.mdl`;
  for a document field, read `components/Docfield/`. The field's `type` decides the filter's
  `dataType` (the filter table in the rules file).
- **Filter values.**
  - A picklist value is the entry name, under `components/Picklist/<picklist>/Picklistentry/`.
  - An object reference is the referenced record's `name__v`.
  - A user is the user's display name (`name__v` on `user__sys`), not the username or email.
  - Record names and user display names are data, not configuration, so get them through the
    vault plugin's query skill.

A value written any other way does not resolve. Vault answers with a "Skipping" warning, and when
no value in the filter resolves, the view is not saved at all.

## Step 4: Draft

Start from the nearest view in the workspace, or from `references/templates/processview-object.mdl`
or `processview-document.mdl`. Copy the folder (or the template), rename it, and change only what
differs. A template's `example_*` names, `start_state__c` and `end_state__c` are placeholders: replace
every one with a name from the workspace. Do not write the XML from memory. A template is one inline statement; in a workspace,
split it into the Step 1 layout: the `settings({...})` body into `settings.xml`, each chart into
`Chart/<chart>/<chart>.mdl`, and `settings(@settings.xml)` plus a `Chart(@Chart/<chart>.mdl)` line
per chart in the parent. Then check the draft against every table in
`references/settings-rules.md`. The ones that catch the most:

- `start_steps` and `end_steps` each hold at least one bare state name, and they differ.
- `timeframe_field` and `timeframe/property` name the same field, `dataType` is `Date`, and the
  `eval` is one of `IN_LAST`, `IN_NEXT`, `BETWEEN`, `GT` in its own value shape.
- Every enum value is in the allowed set: unit, orientation, bin width, interval, calculation
  type, timeframe unit, chart type.
- Between 1 and 10 metrics, each from the accepted list, with `cycle_time_target` set when a
  `quality_completed_*` metric is used.
- `export_columns` holds the process's base set, the timeframe field and every chart's grouping
  field as `required=true`, any extras as `required=false`, each with the `type` its kind needs.
- The two `__sys` charts stay unless the user asks otherwise.
- A custom chart is `line`, `column` or `bar`, never `volume` or `combination`.
- A chart's `metrics` is one comma-separated string.
- A chart's `sort_field` is `grouping.<its grouping field>` or `metric.<one of its metrics>`.
- There are no more than seven custom charts.
- `subprocess` values are in the order Vault stores them, alphabetical, so the second preview
  stays clean.

For a clone, also give the checklist:
- name and label
- process and subprocesses
- steps and calculation
- timeframe
- filters
- metrics and target
- charts
- export columns
- owners, editors and viewers

For each one, say whether it changed or was kept.

## Step 5: Plan and hand off

Stop and show the plan: which folder is new or changed, what the view will measure in one
sentence, and the traps that apply to this draft. On a document lifecycle, the one sentence says
that each steady-state version is measured separately. Nothing is pushed before the user says go. Then
the mdl skill runs it through the workspace loop. The view is one component, so ordering is not
an issue. With the CLI, the files are already in the pulled workspace, so the next push from there
picks them up. Say so, so it is not pushed by accident.

## Step 6: Verify

A success response does not mean the view runs.

Say all of the following in the plan, before the go, as well as after the push:

- After the push, the second `push --dry-run` must show no changes. If it lists the view as
  `MODIFIED`, Vault corrected something. Pull, show the diff, and explain it from the rules file's
  readback table. The usual cause is an export column Vault added or flipped, which means the
  draft's column set was wrong.
- A "Skipping" warning (`BPV-WREC-II-22003`, `22004` or `22006`) means a filter value did not
  resolve, and it counts as a failure. When no value in a filter resolves, Vault saves nothing,
  and the response is a warning, not an error. Check that the view exists before reading it back.
- Read the view back and compare it with the draft. The changes listed in the rules file's
  readback table are normal and are not a reason to resend.
- Nothing runs a Process View except Process Monitor itself. There is no API or CLI command for
  it, so a success response proves only that Vault saved the view. Say so, and ask the user to open
  the view in Process Monitor and confirm it runs. Some mistakes save cleanly and fail only there;
  the rules file's "Accepted, then fails later" table lists them.

## Handoffs

| Need | Skill |
|---|---|
| Pull or refresh the workspace | config |
| Plan, push, run, drop or read back a view | mdl |
| Record names, user display names, a count to sanity-check a view | vault plugin, query |
| A Vault report or dashboard | reporting |
