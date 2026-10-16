# Report, report type and dashboard files

## What each file carries

| File | Holds |
|---|---|
| Report `<name>.mdl` | label, active, description, format, content_type, report_type, editors, viewers, owners, class, excel_template, fit_pdf_to_page, tags, include_full_longtext, include_full_richtext, include_richtext_html_tags, and pointers `definition(@definition.xml)` and `layout(@layout.xml)` |
| `definition.xml` | root `query` in the `vrp` namespace: `querySettings`, `select` with `selectItem`, `where` with `whereItem`, `group` with `groupItem`, `order` with `orderItem`, and aggregates, which are a `selectItem type="FUNCTION"` inside `groupItem/function`: SUM and AVG are the proven forms; the other function actions take the same shape, each with a `children` property. A tabular report never carries a bare COUNT item, because the group header counts. Matrix cells use the bare COUNT |
| `layout.xml` | root `reportFormat`: `columns` with `column`, `relationshipObjectJoinTypes`, and `settings` |
| Reporttype `<name>.mdl` | label, active, description, `primary_object`, primary_objects, class, and `configuration(@configuration.xml)` |
| `configuration.xml` | root `reportType`. A bare root for a simple type. A Standard type adds `upRelationships` or `downRelationships`. A MultiPass type adds `joinRelationships` between reports |
| Dashboard `<name>.mdl` | label, active, description, editors, viewers, owners, tags, and `dashboard_markup(@dashboard_markup.xml)` |
| `dashboard_markup.xml` | `dashboard`, then `chart` elements. Each chart has `position`, `format`, `series`, `grouping`, `range`, and `tableChartReportColumns` with `reportColumn`. A chart names its report as `report="Report.<name>"` |

Property paths such as `glossary_definition__sys.name__v` usually appear inside the XML as the
object name, a dot, then the field. MultiPass keys put the report name first, and a relationship
hop is the relationship key with the `.` replaced by `___`, then the target field (`purchase_order__c___vendor__c.region__c`), for both up and down relationships. The same path shows up in `definition.xml` and in the matching
`column key` in `layout.xml`.

## Reading a definition

- `selectItem` is a column. `property` is the path.
- `whereItem` is a filter. `action` is the operator, `leftValue` the path, `rightValue` the
  value, and `runtime="true"` marks a prompt the user answers when the report runs.
- `groupItem` and `orderItem` carry grouping and sort order.
- An empty `report_type()` with `content_type('Document')` is a document report built on documents directly. The `docType` in `layout.xml` appears on object reports too, so do not read the report's scope from it.

## Clone checklist

Mark each item changed or kept in the draft.

| Item | Where |
|---|---|
| New component name | folder name, `.mdl` file name, and the name after `RECREATE Report` |
| label | `.mdl` |
| description | `.mdl` |
| report_type | `.mdl`, `Reporttype.<name>`. Not editable once the report exists, and neither is `class`, so pick the type before the first push |
| class | `.mdl`. `Standard` opens from Reports. `View` is a report view, hidden from the Reports list. Fixed after create, so a wrong class means DROP and RECREATE |
| Property paths | `definition.xml` and `layout.xml`. If the report type changes, every path changes with it |
| Filters | `whereItem` entries in `definition.xml`, including runtime flags |
| Columns | `selectItem` in `definition.xml` and the matching `column` in `layout.xml`, in the same order |
| Grouping and sort | `groupItem` and `orderItem` |
| owners, viewers, editors | `.mdl`. Group and user references are copied as written, so confirm they exist in the target |
| active | not editable on Report. Do not offer to inactivate |
| Dashboard chart references | for a dashboard clone, each `report="Report.<name>"` |

## What a clone push looks like

| Step | What happens |
|---|---|
| Source | `GET /api/mdl/components/{Type}.{name}` with no version in the path returns the RECREATE form, XML inline as `definition({...})` and `layout({...})` |
| Edit | Change only the name after `RECREATE` and the label. Same report type, so the property paths stay valid |
| Push | `vault mdl "<statement>"` runs the text as-is and answers `MDL executed.` with no warnings. A Dashboard clone behaves the same |
| Readback | Definition, layout and dashboard markup match what was sent. Vault adds the executing user to `owners`, which is expected and not a drift |

RETRIEVE is not an MDL verb and fails with GEN-F-402. Reads go through the GET above.

## Limits

- ALTER is not supported on Report. A change is RECREATE, DROP or RENAME.
- `active` cannot be edited on a Report, so a report is dropped or left alone.
- A Report that a Reporttype references cannot be dropped, and the failure is
  `RPT-FDRO-II-33023`.
- A Report that a dashboard uses cannot be dropped either. The failure is
  `DAB-FDRO-ONA-40001`, so the dashboard goes first.
- MDL accepts XML that the report cannot run, such as a bare COUNT item in a tabular group or a
  filter value that never resolved. Run the report to verify.
- A dashboard whose non-featured charts plus spacers are not a multiple of its column count
  loses charts silently, and the response still says success. Featured charts sit outside the
  grid and are not counted. Count every chart sent, featured or not, on readback.
- Reporttype create and alter are supported. Every alter sets `active` to false unless the statement carries `active(true)`. An alter that changes relationships is refused once
  reports use the type.
- No REST endpoint runs a report and the CLI has no report command.
- `report__sys` is queryable and holds report records. What else it holds is not confirmed.
