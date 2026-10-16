# Authoring rules for a new report type, report and dashboard

The templates in `templates/` follow every rule here. Copy a template, change names and
paths, and check the result against these tables before the plan.

## MDL verbs

| Component | Verb to use | Notes |
|---|---|---|
| Reporttype | `RECREATE` | `CREATE` and `ALTER` also work. Every `ALTER` sets `active` to false, even one that only changes the description, so include `active(true)` and compare `active` on readback. An `ALTER` that changes the relationships while any report uses the type is refused with `GEN-FALT-ONA-1017`, "Component is in-use" |
| Report | `RECREATE` | `CREATE` and `ALTER` are rejected with the generic error below, whatever the docs say. A change is another `RECREATE`, a `DROP` or a `RENAME`. The statement carries `class('Standard')` |
| Dashboard | `RECREATE` | `CREATE` is rejected with the generic error. A second `RECREATE` replaces the whole markup. Charts left out are removed, and a chart whose `name` matches keeps its identity |

`RECREATE` creates the component when the name is new and replaces it when the name exists.

| Drop | Rule |
|---|---|
| Order | Dashboards first, then reports, then the report type, one statement each |
| Report type in use | Refused with `RPT-FREC-ONA-34001`, which names how many reports use it |
| Report a dashboard uses | Refused with `DAB-FDRO-ONA-40001`, `Cannot DROP [report] since it is referenced by dashboard(s) [dashboard]`, so the dashboard goes first |
| Name that does not exist | Fails with the generic error, which is what a retry after a partial cleanup hits for names already gone |

## Report definition

The `definition` XML has root `vrp:query` with `xmlns:vrp="VaultReport"`. A wrong namespace
fails with the generic error.

| Rule | Detail |
|---|---|
| Four sections | `select`, `where`, `group` and `order` are always present. Write `<vrp:where />` or `<vrp:group />` when there is nothing in them |
| `querySettings` | Optional. The templates carry `includeAllVersions="false"` |
| `selectItem` | `type` (`PROPERTY`, `FUNCTION`), `action` (`NONE` for a property), `property` (the path). `display` is optional on a function |
| `whereItem` | `type`, `action`, `leftValue`, `rightValue`, `runtime`. Filters are joined by a separator row, `action="NONE" leftValue="" rightValue="" type="AND"` (or `OR`) |
| `groupItem` | `index`, `name` (the path), `groupBySortOn` (the same path), `groupBySortOrder` (`ASCENDING` or `DESCENDING`). A sort field with no order fails |
| Date grouping | Add `period` to the `groupItem` on a date field: `YEAR`, `QUARTER`, `MONTH`, `WEEK` or `DAY` |
| Group within group | Two `groupItem`, the outer at index 0 and the inner at index 1, with an aggregate on each. The report then shows the total per inner group and per outer group |
| `orderItem` | `index`, `name`, `sortDirection` |
| Aggregate | A `selectItem type="FUNCTION"` with `action` and `property` both set to the function name (`SUM`, `AVG`), and the numeric field as a `PROPERTY` item under `children`. It sits in the group's `function` element |
| Several aggregates | Several function items in one `function` is fine, for example AVG and SUM, each with its own `children` |
| Tabular and COUNT | A tabular report never carries a bare `COUNT` item. The group header counts the records on its own, and the item makes the report fail when it runs |
| Matrix | `format('matrix')` needs two or more `groupItem` (index 0 is the columns, index 1 the rows). Each carries the bare item `type="FUNCTION" action="COUNT" property="COUNT" display="NUMBER"` as the cell calculation. One group fails with `GEN-FREC-ONA-1232` |
| Changing format | Switching tabular and matrix on an existing report is refused while a dashboard shows it |
| `format` | `tabular` or `matrix`, lowercase |
| `content_type` | `Object` for a report on a custom object |
| `class` | Always `Standard`, so the report opens from Reports. A `View` is a report view, a data source for other reports that is hidden from the Reports list, and this skill does not write one. It is fixed after create, so a wrong class means `DROP` and `RECREATE` |
| `report_type` | `Reporttype.<name>`. It cannot be changed once the report exists |

### Property paths

| Field is on | Path | Example |
|---|---|---|
| The primary object | `object.field` | `purchase_order__c.total__c` |
| A related object, up or down | `object___reffield.field` | `purchase_order__c___vendor__c.region__c` (up), `purchase_order__c___vendor__c.total__c` (down, from a `vendor__c` report) |
| A document report | `documents.field` | `documents.status__v` |

The hop is the relationship key with the `.` replaced by `___`, then the target field. The
key names the object that holds the reference field, so up and down build the same way. A report
on `vendor__c` with `<vrt:downRelationship key="purchase_order__c.vendor__c" />` reaches the
purchase order total as `purchase_order__c___vendor__c.total__c`. The path alone does not say
which way it points. The report type does. An unknown path fails the statement with `GEN-FREC-II-1035` naming the path.

### Filter values

| Field kind | `rightValue` | Also set |
|---|---|---|
| Lifecycle state | The state label, `Approved` | `labelValues` with the same label, `labelLanguage="en"` |
| Picklist | The value name, `medium__v` | Nothing |
| Document status | The status API name | `labelValues` with the label |
| Object reference | The record names | Nothing |
| Several values | Joined with `;;;`, `a__c;;;b__c` | |
| Relative date (`INLASTDAYS`, `INNEXTDAYS` and the week, month, quarter and year forms) | The `rightValue` for the `INLAST*` and `INNEXT*` actions is a count of days, for any unit | |
| `BETWEEN` | `start;;;stop`, dates as `yyyy-MM-dd` | |
| Boolean | `true` or `false`, lowercase | Nothing |
| Runtime prompt | Empty `rightValue` | `runtime="true"` |
| `ISNULL`, `ISNOTNULL` | Ignored | |
| Field to field (`F2F__EQ` and the like) | The other field's path | |

Apart from `ISNULL`, `ISNOTNULL` and a runtime prompt, a filter needs a `rightValue` that Vault
can look up, or the statement fails. Dates are not checked until the report runs.

Passing that check does not mean the filter works. A lifecycle state written as its API name
is accepted, then the filter shows "is not equal to" with no value and the report fails when it
runs. A filter value that this table cannot derive from metadata is checked by running the
report.

### Layout

The `layout` XML has root `vrp:reportFormat` with `xmlns:vrp="VaultReport"`.

| Rule | Detail |
|---|---|
| Header attributes | `joinObjectType="NONE" generateComponentEvents="true"` |
| Fixed children | `docType` (`base_document__v`), `targetDocumentAlias` and `sourceDocumentAlias` (`default`), `enableFilterAliases` (`false`) |
| `columns` | One `column` per selected field, in select order. `key` is the same path, `type="documentProperty"`, `index` counts from 0, `columnWidth` is pixels (200 for a name, 155 otherwise) |
| Optional booleans | `combineReportPrompts`, `enableMergeCells`, `mergeCellsWithinView`, `showInactiveFields`, `showInactiveUsers`, `removeLinksFromExport`. Leave them out |
| `relationshipObjectJoinTypes` | Leave it out when authoring. It holds ids that Vault generates |

## Report type

| Rule | Detail |
|---|---|
| Statement | `RECREATE Reporttype <name>__c (label, active, description, primary_object, primary_objects(), configuration({...}), class('Standard'))` |
| `primary_object` | The object name. `primary_objects` stays empty on a Standard type |
| `class` | Set on create and fixed after. `Standard` for an object type |
| Root | `vrt:reportType` with `xmlns:vrt="VaultReporttype"` |
| Up (parent) | `<vrt:upRelationships><vrt:upRelationship key="object.reffield" /></vrt:upRelationships>` |
| Down (child) | `<vrt:downRelationships><vrt:downRelationship key="child.reffield" /></vrt:downRelationships>`, and a child may nest `upRelationships` for its own parent |
| `key` | The object that holds the reference field, a dot, the reference field. Up from the primary: `purchase_order__c.vendor__c`. Down: the child object and its field that points at the primary |
| `keyRef` | Leave it out. Vault generates it |
| `joinType` | Optional, `inner` or `outer`. Leave it out |
| Limits | Label 65 characters, description 255 |
| Custom object | A custom object has no report type until one is created. Check the pull or `vault_component__v` first |

An unresolvable key fails with `GEN-FREC-II-1035`. Malformed XML or a bad `joinType` fails
with the generic error.

## Dashboard

The `dashboard_markup` XML has root `vault:dashboard` with `xmlns:vault="VeevaVault"`.
Children are `chart` and `spacer`. Attributes carry no prefix.

| Rule | Detail |
|---|---|
| Layout `type` | `Two-Columns` or `Three-Columns`. `Eighteen-Columns` and `layoutVersion="V2"` are not used |
| Chart count | Count only charts that are not featured, plus spacers. That total must be a multiple of the layout's column count. Three-Columns takes 3, 6 or 9, and Two-Columns takes 2, 4 or 6. Pad the end with `<vault:spacer />`. A chart with `featured="true"` spans the width above the grid and is left out of the count, so a featured count tile plus three charts on Two-Columns needs one spacer |
| Wrong count | Five charts on Three-Columns is accepted and Vault keeps three, silently dropping the rest. One chart alone on Two-Columns and two charts alone on Three-Columns are rejected with the generic error. Two charts on Two-Columns are kept |
| Chart attributes | `name` (ends `__c`), `label`, `type` and `report="Report.<name>"` are required, and a missing one fails with the generic error. `featured` is optional. The templates carry `featured="false"` |
| Chart children | `format` and `series` are required. `table` adds `tableChartReportColumns` |
| Chart `type` | `bar`, `column`, `control`, `gauge`, `line`, `number`, `pareto`, `pie`, `table`, `combination`. The templates cover number, table, column, bar and pie. A gauge works too, with the range rule below. `control`, `line`, `pareto` and `combination` have no worked form here, so copy one from a pulled dashboard |
| `report` | The report must already exist, so the reports run before the dashboard. A missing one fails with `GEN-FREC-II-1004` |
| `grouping key` | Must be a `groupItem name` of that report. Anything else fails with `GEN-FREC-II-1035` naming the key and the value |
| `grouping value` | `AllValues` or empty |
| `series metric` | `RecordCount` for the primary object, or `object.field` for an aggregate field of the report |
| Other forms | `NoGrouping` and `RecordCount.<relationship>` exist in Vault-written dashboards and are not covered by the templates |
| `unitlabel` | Read for number and gauge charts only |
| Number and gauge | Optional `range` children, each with `min`, `max` and `color`. Every child of `format` is read as a range. A gauge needs `min` and `max` on `format` |
| Table chart | `groupsToDisplay` on `format`. Columns are `reportColumn key label property="data"`, and a grouping column's key is the path with a `grp_` prefix |
| Axis charts | `showlegend`, `showvalue`, `xaxislabel`, `yaxislabel`, and `stacking` as `normal` or `percent`. Any other stacking value fails with the generic error |
| Stacked column | A `column` chart with two `grouping` elements and `stacking="normal"` on `format`. The report groups by both keys, and each `grouping key` is one of its `groupItem name` values |
| Pie | `showlegend`, `showcategory`, `showpercentage` |
| Prompts | A report's runtime prompts appear on the dashboard either way, one set per chart, so three charts on a two-prompt report show six filters. `combinePromptsBy` on the `dashboard` element merges them into one set. A Vault can drop the attribute silently on readback. Read the dashboard back after the push, and when it is gone say the prompts still work but stay per chart. Do not tell the user the merged filters exist |
| Many groups | A chart with many groups (roughly more than six) renders partially in a Three-Columns tile with a "Too many to display" marker. Use Two-Columns or a wider slot for it. The expand view always shows every group |
| Axis labels | `xaxislabel` and `yaxislabel` are fixed strings. Name the chart's own grouping field and metric, and change them when a template chart is reused on another grouping |

## Errors

| Error | Means | Check |
|---|---|---|
| `GEN-EREC-UE-801` (the middle letters change with the verb) | Generic. It names no attribute | In order: a bad enum value (`action`, `type`, `format`, `stacking`, a chart type), a missing section (`where`, `group`, `order`, `select`), a missing chart attribute, then the chart count against the layout |
| `GEN-ECRE-UE-801` on `CREATE Report` or `CREATE Dashboard` | The verb is not handled | Use `RECREATE` |
| `GEN-FREC-ONA-1232` | A matrix report has one group | Add a second `groupItem` |
| `GEN-FREC-II-1035` | A path, key or value does not resolve | The text names it. For a dashboard it is the grouping key not in the report |
| `GEN-FREC-II-1004` | A chart's report is missing | Run the report first |
| `RPT-FREC-ONA-34001` | Drop of a report type some reports use | Drop the reports |
| `DAB-FDRO-ONA-40001` | Drop of a report that a dashboard uses | Drop the dashboard first, then the report |
| `GEN-FALT-ONA-1017` | `ALTER Reporttype` that changes relationships while a report uses the type | Drop or re-point the reports first, or create a new type |
| `GEN-EDRO-UE-801` | A `DROP` of a dashboard name that does not exist returns the generic error | Read the component list |

## Accepted, then fails later

| Symptom | Cause |
|---|---|
| Success response, report errors when it runs | A bare `COUNT` item in a tabular group, or a filter value that never resolved |
| Success response, fewer charts than sent | Non-featured chart plus spacer count is not a multiple of the columns. Featured charts are not counted, so a featured tile does not make up the difference. Add a spacer and run it again |
| Success response, filter shows "is not equal to" with no value | A state filter written as the API name instead of the label |
| Success response, report missing from Reports | Check `class`. A `View` is hidden from the list. Drop and recreate it as `Standard` |

## What Vault adds on readback

None of these is an error or a reason to recreate.

| Component | Vault adds or changes |
|---|---|
| Report | `rightValueType="VALUE"` on filters, `showInactiveUsers` in the layout, the executing user in `owners`, and the attributes left out at their defaults (`editors`, `viewers`, `class`, `excel_template`, `fit_pdf_to_page`, `tags` and the `include_*` flags) |
| Dashboard | `labelLanguage` on groupings, gauge range colors rewritten, `showmetrics` and `showgroupings` dropped from a table chart, `format` attributes reordered, the executing user in `owners`, and an empty `tags()` |
| Report type | An XML declaration at the top of the configuration. The rest matches what was sent |
