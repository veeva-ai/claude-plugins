---
name: reporting
version: "0.1.0"
description: >
  For anyone who asks for a Vault report or dashboard: "I need a report of X by Y", "build me a
  report", "a dashboard of open expenses by department", "put that on a dashboard", "with a pie
  chart", "report type for my object". In Vault a report is a component with a report type, a
  definition and a layout, not a VQL query, so when the user says report or dashboard this skill
  owns it even though the data could be fetched with VQL. Use it to build a report type, a report
  or a dashboard from scratch, to draft a clone of one under a new name, to explain what an
  existing report selects, filters, groups and joins, and to trace a report to its report type or
  a dashboard to its reports. Not for a one-off data pull or a VQL question (vault → query).
---

# Vault Reports and Dashboards

Reports, report types and dashboards are components, so a default `vault config pull` brings
them down as files. This skill reads those files, explains them, and drafts changes to them as
MDL. It does not run reports, and it does not write to a Vault. The mdl skill executes what
this skill drafts.

## Scope

- Explaining what an existing report selects, filters, groups and joins, from its pulled files.
- Tracing dependencies: a report to its report type and primary object, a dashboard to the
  reports it shows.
- Drafting a clone of an existing report or dashboard under a new name, with a checklist of
  what to change.
- Drafting a report type for a custom object.
- Building a report type, reports and a dashboard from the templates, up to the user's go.

Out of bounds:

- Report types with more than one primary object. Those are Admin UI work.
- Eighteen-Columns and V2 dashboard layouts. Those are Admin UI work.
- Running a report. No REST endpoint runs one, and the CLI has no report command. Running is
  the Vault UI.
- Report views (`class('View')`). A view without a report that uses it can only be reached by
  link or id, so this skill does not write one.
- Building a report visually. The XML is drafted here from the templates.
- Scheduled or flash report delivery, and who may see a report. Sharing rules are Admin UI work.
- Executing any change. Drafts go to the mdl skill.
- A count or list of report records, or who ran which report. That is a data question for the
  vault plugin's query skill (`report__sys`, `report_usage__sys`). Building, explaining or
  cloning a report stays here.

## Step 1: Find the files

Reports travel through `vault config pull`, mdl and push. If there is no workspace, hand
the pull to the config skill, then come back. Check the files before answering.

```
components/Report/<name>/<name>.mdl
components/Report/<name>/definition.xml
components/Report/<name>/layout.xml
components/Reporttype/<name>/<name>.mdl
components/Reporttype/<name>/configuration.xml
components/Dashboard/<name>/<name>.mdl
components/Dashboard/<name>/dashboard_markup.xml
```

Each type's metadata is in `.vault/metadata/<Type>.json`. To list the reports a Vault has without a
workspace, `vault_component__v` filtered to `component_type__v = 'Report'` returns them. That
lists components. It does not show what a report contains.

## Step 2: Explain a report

Read the report's `.mdl` first, then `definition.xml`, then `layout.xml`. The shape of each file
is in `references/report-definition.md`. In the answer:

1. Name the report type and, from that report type's `.mdl`, its primary object. An empty
   `report_type()` with `content_type('Document')` is a document report built on documents
   directly. The `docType` in `layout.xml` appears on object reports too, so do not read the
   report's scope from it.
2. List what it selects, by property path, in the order the columns appear.
3. State the filters in plain words, one line each, and say which carry
   `runtime`. A filter with `runtime="true"` is a prompt the user answers when the report runs.
4. State how it groups and sorts, and any aggregates.
5. Say how the report type reaches related objects. On a `class('Standard')` type that is
   `upRelationship` and `downRelationship` entries, each keyed `<child_object>.<reference_field>`.
   Only a `class('MultiPass')` type has `joinRelationships`, and those join one report to
   another, so say which reports and whether the join is outer. Only outer has been seen.
6. Give the owners, viewers and editors from the `.mdl`.

Quote the file paths so the user can open them. Do not guess at a property path's meaning. The
common path, like `glossary_definition__sys.name__v`, is the object followed by the field. Other
forms exist (a report name in front on MultiPass types, a `___` hop across a relationship), so
look the field up in the workspace before describing it.

## Step 3: Trace dependencies

The recipes are in `references/dependency-trace.md`. The chain runs one way:

- Report to report type, through the `report_type` attribute in the report's `.mdl`.
- Report type to primary object, through `primary_object` in the report type's `.mdl`.
- Dashboard to reports, through `report="Report.<name>"` in `dashboard_markup.xml`.

To find what depends on a report, search the other direction: every dashboard that names it,
and every report type that still references it matters before a drop. A Report that a Reporttype
references cannot be dropped, and neither can one a dashboard uses, so check before offering one. A MultiPass type names the report in
its `primary_object` and its join keys, and the recipe is in the reference.

Say what the trace covers. It finds references in the pulled files, so it is only as current
as the pull. If the workspace is old, ask the config skill for the freshness check first.

## Step 4: Draft a clone

A clone is a new component, so it is drafted as a RECREATE under a new name. The mdl skill plans
it first. A same-type clone of a Report or a Dashboard with only the name and label changed is
accepted by Vault through MDL. Start from the nearest existing component's own folder, copy it,
and change only what differs. Check the source's `class` first, because pulled files include report
views. A pulled report view is not a clone source for a report the user wants to open from Reports.
Do not retype the XML from memory. A definition with no close
match is built from the templates in Step 6.

When the source is a live component and not a pulled folder, `GET /api/mdl/components/{Type}.{name}`
returns the RECREATE form with the XML inline as `definition({...})` and `layout({...})`. That text
runs as-is through `vault mdl "<statement>"`. RETRIEVE is not an MDL verb. After the push Vault
adds the executing user to `owners`, so expect that one difference on readback.

Draft three things and stop:

1. The new folder with the renamed `.mdl`, `definition.xml` and `layout.xml` (or
   `dashboard_markup.xml`), written in the workspace.
2. The checklist from `references/report-definition.md`, each item marked changed or kept.
3. The handoff line: the mdl skill plans and pushes it after the user says go. The folder now
   sits in the pulled workspace, so the next `vault config push` from there will pick
   it up. Say so, so it is not pushed by accident.

For a dashboard, every `report="Report.<name>"` has to name a report that exists in the
target, or the statement fails. Clone the reports first if the dashboard is meant to show copies.

Adding a column or a filter is an edit to the file, made by the rules in
`references/authoring-rules.md`: the property path has to be valid for the report type, and the
filter value is written the way the filter table says. Column order is `index` and width is
`columnWidth` on the layout columns. The Admin UI is for how the report looks when run, not for
structure.

## Step 5: Draft a report type

A report type for a custom object is plain MDL. The `primary_object` names the object, and a
simple type has a bare root in `configuration.xml`, `<vrt:reportType xmlns:vrt="VaultReporttype" />`.

```
RECREATE Reporttype <name>__c (
  label('<label>'),
  active(true),
  description(),
  primary_object('<object>__c'),
  primary_objects(),
  configuration(@configuration.xml),
  class('<class>')
);
```

Start from `references/templates/reporttype.mdl`. When an existing Standard type in the
workspace is close to what is needed, copy that one instead. Either way, change only the object
and field names, and do not write the XML from memory.

Copy the class value from the nearest existing report type of the same shape. Object types in
the sandbox are all `class('Standard')`. A type that reaches a related object carries a
`downRelationship` for a child or an `upRelationship` for a parent in `configuration.xml`, keyed
`<child_object>.<reference_field>`, for example
`<vrt:downRelationship key="project__c.related_initiative__c" />`. A `joinRelationships` block
belongs to a `class('MultiPass')` type, which joins reports, so do not draft one for an object.

An `ALTER Reporttype` sets `active` to false, so every one carries `active(true)` and the readback
compares `active`. An ALTER that changes relationships is refused with `GEN-FALT-ONA-1017` while
any report uses the type. The user drops or re-points the reports first, or a new type is created.

Veeva's docs call the report type XML proprietary and recommend the Admin UI. Say that in one
line, then draft anyway, and hand execution to the mdl skill.

## Step 6: Build a report type, reports and a dashboard

The rules are in `references/authoring-rules.md` and the starting files are in
`references/templates/`. Draft from the templates and do not retype XML from memory.

1. Confirm the ask: the objects, how the data groups, which filters, which chart kinds. If a
   filter value is not obvious from the field, ask what it should read.
2. Check whether a report type already has the primary object and the relationships the report
   needs. In a pulled workspace, read `primary_object` and `configuration.xml` under
   `components/Reporttype/`. If none fits, say so plainly, because a custom object gets no
   report type by default. Draft the Reporttype from `templates/reporttype.mdl`. It goes first
   in the plan, and the reports are drafted on it.
3. Draft each report from `templates/report-grouped.mdl`. Take paths from the object's own
   MDL, and write each filter value the way the filter table in the rules file says. The report
   is always `class('Standard')`, so it opens from Reports. A report
   the user already has is read from `components/Report/` instead, for its name and its
   group-by paths.
4. Draft the dashboard from `templates/dashboard.mdl`. Every chart names one of the reports
   and groups by one of that report's group-by paths. Count only the charts that are not
   featured, plus the spacers. That total must be a multiple of the column count
   (Three-Columns is 3, Two-Columns is 2), so pad the end with `<vault:spacer />`. A chart
   with `featured="true"` spans the width above the grid and is left out of the count. For
   example, a featured count tile plus three charts on Two-Columns needs one spacer, because
   three charts alone do not divide by two. A count that does not fit is either rejected or loses
   charts while Vault still answers success.
5. Stop and show the plan: the report type, each report, the dashboard, and the order. Nothing
   is pushed before the user says go. Then the mdl skill runs them, report type first, then
   reports, then the dashboard, one statement at a time.
6. Verify, because a success response from MDL does not mean the report runs or that every
   chart landed.
   - Read each component back and compare it to what was sent, `class` included. A report
     that is not `Standard` is hidden from Reports and has to be dropped and recreated. The drift list at the end of
     the rules file is expected and is not a reason to resend.
   - Count the charts in the dashboard readback against the charts sent, featured ones
     included. Fewer back than sent is a failed build, whatever MDL answered. A missing chart
     means the count missed, so add a spacer and run the dashboard again.
   - If the dashboard carried `combinePromptsBy`, check it is still on the readback. A Vault
     can drop it silently. If it is gone, say the report's runtime prompts still work but stay
     per chart, and do not claim the merged filters exist.
   - Get the VQL for one group's total from the query skill, then ask the user to run the
     report in the Vault UI and compare that group's total with it. A report that errors on run
     goes back to the filter values and the aggregate rules first.

A failure at any statement stops the run. Read the error against the errors table in the rules
file and say what landed. A drop of a report that a dashboard uses is refused with
`DAB-FDRO-ONA-40001`, so drop the dashboard first. Fix the failing statement and run it again, since RECREATE replaces a
component that already exists. To remove a partial build instead, drop in the order the rules
file gives.

## Handoffs

| Need | Skill |
|---|---|
| Pull or refresh the workspace, or pull an existing report's XML to copy from | config |
| Plan, push, run, drop or read back a report, report type or dashboard | mdl |
| Check a report's totals against a count, report records, who ran what | vault plugin, query |
| Who can see or edit a report | Admin UI sharing rules |
