# Processview settings and charts: rules

The rules for a `Processview` component: its `settings` XML and its `Chart` subcomponent. The
Vault docs describe `settings` only as an XML string, so these come from probing Vault directly,
on an object lifecycle. Every allowed value below was confirmed against Vault's own validation
messages there. Document lifecycle views were not probed: their rules come from existing views.
"Not covered yet" at the end lists what has not been checked.

**Markers:**

- **Validated**: MDL accepted it and readback stored it as sent.
- **Runs**: it also ran in Process Monitor.
- **Rejected**: MDL refused it, with Vault's message quoted.
- **Observed**: seen in existing views only, not probed.

## MDL statement shape

```
RECREATE Processview <name>__c (
   label('<label>'),
   description(),
   process('Objectlifecycle.<lifecycle>'),
   subprocess('Objecttype.<object>.<type>','Objecttype.<object>.<type>'),
   editors(),
   viewers(),
   owners('User.admin@example.com'),
   settings({<?xml version="1.0" encoding="UTF-8"?>
<settings>…</settings>}),
   Chart cycle_time_trend__sys( … ),
   Chart cycle_time_distribution__sys( … )
);
```

| Attribute | Rule | Status |
|---|---|---|
| Component name | 40 characters or fewer including `__c`. No double underscore before the namespace (`GEN-FREC-II-1027: Invalid component name format`) | Rejected otherwise |
| `label` | Required, **up to 128 characters**. The docs say 60, but 61 is accepted and 129 fails with `GEN-FREC-II-1009` | Validated |
| `description` | Optional, up to 255 characters | Observed |
| `process` | `Objectlifecycle.<name>`, `Doclifecycle.<name>`, or `<object>.<process>` for a standard object process. Unknown values fail with `Invalid process value[...]` | Validated |
| `subprocess` | Multi-value, one quoted value per entry: `Objecttype.<object>.<type>` or `Doctype.<type>[.<subtype>[.<classification>]]`. Each is checked against the lifecycle (`Invalid subprocess value[...]`). **Vault stores them in its own order** (alphabetical in every case seen). Empty `subprocess()` measures every type on the lifecycle, as the UI does when no subprocess is picked; every probe view was written this way and ran. Vault Help allows up to six subprocesses, 255 characters in all; that limit was not probed | Validated |
| `owners` | **At least one** `User.<username>` or `Group.<name>`. With none: `Cannot delete all manually assigned users and groups from the owners role` | Rejected otherwise |
| `editors`, `viewers` | Optional, same format as `owners` | Validated |
| `settings` | **Required in practice**. Omitting it raises `GEN-EREC-UE-801`. 32,000-character maximum | Rejected otherwise |

Write every attribute, with empty `attr()` for unset ones, as Vault does on readback.

### Vault CLI workspace layout

```
components/Processview/<name>/<name>.mdl                    settings(@settings.xml), Chart(@Chart/<chart>.mdl), …
components/Processview/<name>/settings.xml                  XML declaration, indented, no padding inside text nodes
components/Processview/<name>/Chart/<chart>/<chart>.mdl     Chart <chart> ( … );   (no RECREATE prefix)
```

The parent `.mdl` references each chart as `@Chart/<chart>.mdl`, even though the file sits one
folder deeper. Copy that form as pulled.

## `settings` element order and requiredness

Vault **reorders elements into this order on save** and **silently drops unknown elements**
(Validated). Write them in this order so the readback diff stays clean.

```
cycle_time_target            optional
cycle_time_unit              REQUIRED
graph_orientation            REQUIRED
distribution_bin_width       REQUIRED
times_series_interval        REQUIRED
start_steps                  needed in practice
start_step_calculation_type  REQUIRED
end_steps                    needed in practice
end_step_calculation_type    REQUIRED
timeframe_field              REQUIRED
timeframe                    REQUIRED
cycle_time_step_calculation  optional (Vault stores it before filters)
filters                      optional
metrics                      needed in practice
export_columns               optional (Vault fills the required set)
```

- **REQUIRED** means MDL rejects the view without it. A missing scalar fails with `Invalid ... value[null]`; a missing calculation type or timeframe raises `GEN-EREC-UE-801`.
- **Needed in practice** means MDL accepts the view without it, but the view then returns no data or fails at runtime (see "Accepted, then fails later").

## Scalar values (Validated; Runs where marked)

| Element | Allowed | Notes |
|---|---|---|
| `cycle_time_unit` | `days`, `hours` | Case-sensitive (`HOURS` is rejected). **Runs.** |
| `graph_orientation` | `horizontal`, `vertical` | The Process Graph direction. Vault maps these to `RIGHT` and `DOWN` internally. |
| `distribution_bin_width` | `auto`, `1`, `2`, `5`, `10`, `25`, `50` | Integers only; anything else fails with `Bucket size: X not supported`. **Runs.** |
| `times_series_interval` | `day`, `week`, `month`, `auto` | Singular; `weeks` is rejected. Note the spelling `times_`. **Runs.** |
| `start_step_calculation_type`, `end_step_calculation_type` | `first_time_in`, `last_time_in` | Either value on either side. **Runs.** |
| `cycle_time_target` | Decimal > 0, in `cycle_time_unit` | Added in 26R1. `10` is stored as `10.0`. `0` or a negative value fails with `Cycle time target must be greater than 0.`; a non-number raises `GEN-EREC-UE-801` |

## Steps

```xml
<start_steps><step>new_state__c</step></start_steps>
<end_steps><step>closed_state__c</step><step>cancelled_state__c</step></end_steps>
```

- Each `<step>` holds a bare state name (or, for a standard object process, a process step name such as `step_1__v`). The qualified form `Objectlifecyclestate.<name>` is rejected.
- Unknown values fail with `Invalid start_steps value[...]`.
- **Runs:** more than one `<step>` on either side.
- **Rejected:** the same value as both start and end (`Invalid duplicate Start and End step value`).
- The legacy singular `<start_step>x</start_step>` is still accepted and stored as `<start_steps><step>x</step></start_steps>`. Write the plural form.

## Timeframe

```xml
<timeframe_field>pm_entered_end_date__sys</timeframe_field>
<timeframe>
  <eval>IN_LAST</eval>
  <property>pm_entered_end_date__sys</property>
  <dataType>Date</dataType>
  <values><values>3</values><values>MONTHS</values></values>
</timeframe>
```

| `eval` | Value shape | Status | UI label |
|---|---|---|---|
| `IN_LAST` | `<values><values>N</values><values>UNIT</values></values>`, N ≥ 1, UNIT ∈ `DAYS`, `WEEKS`, `MONTHS` | Runs | Is in the last |
| `IN_NEXT` | as `IN_LAST` | Runs | Is in the next |
| `BETWEEN` | `<values><values>YYYY-MM-DD</values><values>YYYY-MM-DD</values></values>` | Runs | Is in the range |
| `GT` | `<value>YYYY-MM-DD</value>` (single `value`) | Runs | Is after |

- **Rejected evals:** `LT`, `GTE`, `LTE`, `EQ`, `NE`, `ISNULL`, `ON`, `BEFORE`, `AFTER`.
- **Rejected values:** units `YEARS`, `QUARTERS` and `HOURS`; lowercase `months`; `MM/DD/YYYY` dates; a reversed pair (`[MONTHS, 3]`).
- Whether `BETWEEN` includes its end date has not been checked. To be sure, confirm in Process Monitor that a record on the end date is counted.
- `dataType` is always `Date`, including for DateTime fields and pm dates. `String` and `DateTime` are rejected.
- `property` must equal `timeframe_field` (see "Accepted, then fails later").
- The fields that run are `pm_entered_end_date__sys` (the UI default), `pm_entered_start_date__sys`, `created_date__v`, and `document_creation_date__v` for documents.
- A custom Date field is accepted and runs, but the UI's Date picker shows it blank.

## Filters

```xml
<filters>
  <field><eval>EQ</eval><property>severity__c</property><dataType>Picklist</dataType><values><values>high__c</values></values></field>
  <field><eval>ISNOTNULL</eval><property>name__v</property><dataType>String</dataType></field>
</filters>
```

Multiple `<field>` elements are ANDed. Every row below ran in Process Monitor.

| Field type | `dataType` | Operators (Validated) | Value |
|---|---|---|---|
| Text | `String` | `EQ`, `NE`, `STARTSWITH`, `ISNULL`, `ISNOTNULL` | `<value>text</value>` |
| Picklist | `Picklist` | `EQ`, `NE`, `IN`, `ISNULL`, `ISNOTNULL` | `<values><values>entry_name__c</values>…</values>`: entry **names**, not labels |
| Object reference | `ObjectReference.Object` | `EQ`, `NE`, `IN`, `ISNULL`, `ISNOTNULL` | `<values><values>Record Name</values></values>`: the record's **`name__v`**, not its ID |
| User reference | `ObjectReference.User` | `EQ` | The user's **display name** (`name__v` on `user__sys`), not the username or ID |
| Date | `Date` | `EQ`, `GT`, `LT`, `BETWEEN`, `IN_LAST`, `ISNULL` | `<value>YYYY-MM-DD</value>`; `BETWEEN` and `IN_LAST` use `values` as in the timeframe |
| DateTime | `DateTime` | `EQ` | `<value>2025-01-01T00:00:00.000Z</value>` |
| Boolean | `Boolean` | `EQ` | `<value>true</value>` |
| Lifecycle state (`state__v`) | `Component` | `EQ` | `<values><values>new_state__c</values></values>` |

**Rejected:**

- `IN`, `NOTIN`, `CONTAINS`, `LIKE` and `GT` on `String`
- `GTE` on `Date`
- unknown operators
- unknown fields (`Field "x" is not available for the current process and subprocess selection`)

## Metrics

- 1–10 `<metric>Processmetric.<name></metric>` elements. 11 fails with `Maximum of 10 metric values exceeded`.
- The prefix is required, and duplicates are rejected.
- Order is generally kept as written.

| Group | Accepted (`__sys`) | Rejected (`Invalid metric value[...]`) |
|---|---|---|
| Cycle time | `cycle_time_average`, `cycle_time_median`, `cycle_time_1st_quartile`, `cycle_time_3rd_quartile`, `cycle_time_target` | `cycle_time_company_average`, `cycle_time_company_median` |
| Volume | `volume_records_created`, `volume_entered_start_state`, `volume_entered_start_state_per_{day,week,month}`, `volume_entered_end_state`, `volume_entered_end_state_per_{day,week,month}` | `volume_entered_state`, `volume_entered_state_percent` |
| Quality | `quality_first_time_right_percent`, `quality_first_time_right_total`, `quality_rework_total`, `quality_variant_total`, `quality_completed_on_time_{total,percent}`, `quality_completed_late_{total,percent}` | `quality_rework_time`, `quality_preferred_variant_{total,percent}` |

- The rejected metrics are in the `Processmetric` catalog but not accepted on a custom view. Open question: are they reserved for Veeva-owned standard views?
- `quality_completed_*` is accepted without a target but computes only when `cycle_time_target` is set.
- Added in 26R1: `cycle_time_target`, `volume_records_created`, the per-day, per-week and per-month volume metrics, `quality_completed_*` and `quality_variant_total`. A Vault on an earlier release does not have them.
- The UI default set, in order:
  1. `cycle_time_average`
  2. `cycle_time_median`
  3. `cycle_time_1st_quartile`
  4. `cycle_time_3rd_quartile`
  5. `volume_entered_start_state`
  6. `volume_entered_end_state`
  7. `quality_first_time_right_percent`
  8. `quality_variant_total`

## Export columns

```xml
<export_columns>
  <column><name>name__v</name><type>field</type><required>true</required></column>
  <column><name>pm_cycle_time__sys</name><type>pm_process_metric__sys</type><required>true</required></column>
</export_columns>
```

Vault owns the `required` flag and recalculates it on every save (Validated):

> `required=true` ⇔ column ∈ base set ∪ { `timeframe_field` } ∪ { every chart's `grouping_field` }

| Process | Base set |
|---|---|
| Object lifecycle, standard object process | `name__v`, `modified_date__v`, `pm_cycle_time__sys`, `pm_entered_end_date__sys`, `pm_variant_id__sys` |
| Document lifecycle | `name__v`, `created_by__v`, `id`, `pm_effective_major_version__sys`, `version_modified_date__v`, `major_version_number__v`, `minor_version_number__v`, `status__v`, `type__v`, `pm_cycle_time__sys`, `pm_entered_end_date__sys`, `pm_variant_id__sys` |

Column types:

- `pm_process_metric__sys` for `pm_cycle_time__sys`, `pm_entered_start_date__sys`, `pm_entered_end_date__sys`, `pm_variant_id__sys` and `pm_variant_sequence__sys`.
- `field` for everything else, **including `pm_effective_major_version__sys`**, which is valid on document processes only.

How Vault handles columns that don't follow the rule:

| Input | Vault |
|---|---|
| A required column is missing | Adds it with `WARNING BPV-WREC-II-22013: Adding required export_columns field[...]` |
| No `export_columns` at all | Builds the required set |
| A user column marked `true`, or a base column marked `false` | Flips it |
| `name__v` not first | Moves it first (`WARNING BPV-WREC-II-22014`) |
| A duplicate column | Drops it silently |
| A wrong `type` | **Drops the column silently** |
| An unknown field or a relationship path | Rejects the statement |

## `cycle_time_step_calculation`

```xml
<cycle_time_step_calculation>
  <include_steps>false</include_steps>
  <matching_steps><step>on_hold_state__c</step></matching_steps>
</cycle_time_step_calculation>
```

- **Runs.** `include_steps=false` excludes time spent in the listed steps from cycle time.
- **Runs.** `include_steps=true` counts only time in the listed steps.
- Steps are validated (`Invalid value [...] in [matching_steps]`).
- An empty `matching_steps` makes Vault drop the element.

## `Chart` subcomponent

```
Chart severity_cycle_time__c(
   label('Cycle Time by Severity'),
   type('column'),
   metrics('Processmetric.cycle_time_average__sys'),
   grouping_field('severity__c'),
   second_grouping(),
   sort('desc'),
   sort_field('metric.cycle_time_average__sys'),
   order(3)
)
```

| Attribute | Rule |
|---|---|
| `label` | Required, **at most 60 characters** |
| `type` | **Required.** Allowed: `bar`, `column`, `distribution`, `line`, `combination`, `volume`. **Runs:** `line`, `column`, and `bar` (drawn as vertical bars). `volume` and `combination` are accepted but **render empty** with every metric tried |
| `metrics` | **One quoted, comma-separated string.** A list fails with `GEN-FADDS-II-1041: Single value is expected`. Values use the `Processmetric.` prefix and are validated. A chart may use a metric that isn't in `<metrics>` |
| `grouping_field` | Validated against the process. **Runs:** Date, picklist, object reference, user reference, Boolean, `state__v`, `pm_entered_start_date__sys`. **Rejected:** `name__v`, `pm_variant_id__sys`, and `pm_cycle_time__sys` on a custom chart. The grouping field becomes a required export column |
| `sort` | `asc` or `desc`. **Runs** |
| `sort_field` | `grouping.<this chart's grouping_field>` or `metric.<a metric of this chart, no prefix>`. Anything else fails with `Invalid sort_field value` |
| `second_grouping` | Accepted and stored, **ignored at runtime**. Write `second_grouping()` |
| `order` | Optional. Missing values are assigned; duplicates are renumbered. Write 1..n |
| Custom charts | At most 7, besides the two `__sys` defaults (`Maximum of 7 custom charts exceeded`) |

Default charts. Without them, the view has no charts.

| Name | `type` | `metrics` | `grouping_field` | `sort_field` | `order` |
|---|---|---|---|---|---|
| `cycle_time_trend__sys` | `line` | the four cycle-time metrics | `pm_entered_end_date__sys` | `grouping.pm_entered_end_date__sys` | 1 |
| `cycle_time_distribution__sys` | `distribution` | the same | `pm_cycle_time__sys` | `grouping.pm_cycle_time__sys` | 2 |

## Accepted, then fails later

| Symptom | Cause |
|---|---|
| Statement returns `WARNING … Skipping` and **no view exists** | No value in a filter resolved: a picklist label, a record ID, or a username. Happened in 7 of 7 cases. When some values in the filter still resolve, the view is saved without the bad ones |
| View runs with no data | `start_steps` or `end_steps` missing or empty |
| View never runs; the Date picker is blank | `timeframe_field` differs from `timeframe/property` |
| "Process Monitor hit an unexpected error … Error executing query" | No metrics, or a filter `dataType` that doesn't match the field type (a user reference typed `ObjectReference.Object`). MDL doesn't check `dataType` against the field |
| Custom chart renders empty | `type('volume')` or `type('combination')` |
| Chart identical with or without `second_grouping` | Ignored at runtime |

## CLI round trip

| Check | Result |
|---|---|
| New view from a copied folder | Push stores it exactly as written. Both dry runs report unchanged, and a re-pull leaves the files byte-identical |
| Narrowing `subprocess` | Applied. Values come back in Vault's order |
| Adding or removing a chart folder (plus its line in the parent) | Applied. A removal is an update to the view, not a delete prompt |
| A change Vault corrects (an export column added or its `required` flipped) | The second `push --dry-run` reports `MODIFIED`, and a re-pull shows the correction |
| Deleting the view's folder | `push --dry-run` lists `DELETED Processview.<name>`, and a real push stops for confirmation |
| `.vdxignore` with `components/*` then `!components/Processview/` | A Process-View-only workspace. Push previews list no deletes for the ignored types |

## What Vault changes on readback

These differences are not errors:

- element order
- `subprocess` order
- `cycle_time_target` as a decimal
- `required` flags recalculated
- the missing required export columns added
- `name__v` moved first
- unknown elements dropped
- legacy `start_step`/`end_step` converted to the plural form
- chart `order` filled in or renumbered

## Not covered yet

- Document lifecycle views were not probed; their rules come from five existing views.
- Only one standard object process view has been seen.
- The minimum Vault release for `Processview` MDL has not been confirmed.
- `NOTIN` on picklists, and operators other than `EQ` on user references.
- Whether `BETWEEN` includes its end date.
- The six-subprocess limit from Vault Help.
- On the workspace path, how the CLI reports a push where Vault saved nothing because no filter value resolved. The second preview listing the view as `ADDED` follows from the preview comparing against the Vault.
