# Page Layout Markup Reference

This file documents the XML structure of the `page_markup` XMLString attribute on a `Pagelayout`
MDL component. Because `page_markup` is a single XMLString, there is no section-level subcomponent
to target: any structural change requires retrieving the current markup, editing it, and
resubmitting the entire XML value, whether via `RECREATE Pagelayout` or `ALTER Pagelayout`.

`RECREATE` replaces the whole component (all attributes). `ALTER` updates only `page_markup` (or
other named attributes) and leaves the rest untouched, but you must still supply the full XML
string for `page_markup` since Vault has no finer-grained subcomponent to modify.

The XML is wrapped in `{ }` per the XMLString syntax rule:

```
RECREATE Pagelayout my_layout__c (
    label('My Layout'),
    active(true),
    page_markup({
        <vault:page xmlns:vault="VeevaVault" controller="Objecttype.my_object__c.base__v">
            ...
        </vault:page>
    })
);
```

Or, if you only want to change one attribute for an existing Pagelayout:

```
ALTER Pagelayout my_layout__c (
    page_markup({
        <vault:page xmlns:vault="VeevaVault" controller="Objecttype.my_object__c.base__v">
            ...
        </vault:page>
    })
);
```

Vault validates XML well-formedness only, not schema correctness: an invalid element
name or attribute silently fails (or ignored) to render rather than returning an MDL error.

---

## Root element

Two root element forms exist depending on whether the layout has one page or multiple pages.

### Single-page layout: `vault:page` as root

```xml
<vault:page xmlns:vault="VeevaVault" controller="Objecttype.my_object__c.base__v">
    <vault:section ...>...</vault:section>
</vault:page>
```

### Multi-page layout: `vault:pages` as root

```xml
<vault:pages xmlns:vault="VeevaVault" controller="Objecttype.my_object__c.base__v">
    <vault:page label="Page 1" name="page1__c">
        <vault:section ...>...</vault:section>
    </vault:page>
    <vault:page label="Page 2" name="page2__c">
        <vault:section ...>...</vault:section>
    </vault:page>
</vault:pages>
```

When using `vault:pages`, the `controller` attribute moves up to `vault:pages`. Each child
`vault:page` carries a display `label` (the page label) and an API `name`.

### Root element attributes

| Attribute | On | Required | Description |
|---|---|---|---|
| `xmlns:vault` | Root | Yes | Always `"VeevaVault"`. Must be present on the root element. |
| `controller` | Root | Yes | The object or object type this layout governs. Two forms: `"Object.{object_name}"` (applies to all types) or `"Objecttype.{object_name}.{type_name}"` (type-specific). |
| `template` | `vault:page` only | No | `"record_detail__sys"` (default) renders a standard field-and-section layout. Omit for all standard layouts. |

### `vault:page` child attributes (multi-page only)

| Attribute | Required | Description |
|---|---|---|
| `label` | Yes | Display string shown as the page label. |
| `name` | Yes | API name for the page. Must carry a namespace suffix matching the layout's package. |

---

## Sections: `vault:section`

Each `vault:section` is a collapsible panel on the record detail page.

```xml
<vault:section title="Details" name="details__c">
    ...
</vault:section>
```

| Attribute | Required | Description |
|---|---|---|
| `title` | Yes | Display label shown as the section header. Plain string (localization is handled outside the page markup XML). |
| `name` | Yes | API name for the section. Must carry a namespace suffix (`__c`, `__v`, `__rim`, etc.) matching the package the layout belongs to. |
| `helpContent` | No | Help text string shown as a tooltip on the section header. Max 500 characters. |

Each section contains exactly one content element: one `vault:detailform`, one
`vault:relatedObject`, one `vault:uiwidget`, one `vault:wftimeline`, or one `vault:attachments`.
Multiple content elements inside a single section are not supported.

Optionally, one or more `vault:sectionState` children may precede the content element to restrict
which lifecycle states the section is visible in (see [Lifecycle state visibility](#lifecycle-state-visibility-vaultsectionstate) below).

---

## Lifecycle state visibility: `vault:sectionState`

To show a section only in specific lifecycle states, place one `vault:sectionState` child per
state inside `vault:section`, before the content element. When no `vault:sectionState` children
are present, the section is visible in all states.

```xml
<vault:section title="Attachments" name="attachments__v" helpContent="Attach files here.">
    <vault:sectionState state="submission_lifecycle_lifecycle__c.ha_received_state__c" />
    <vault:sectionState state="submission_lifecycle_lifecycle__c.in_progress_state__c" />
    <vault:attachments />
</vault:section>
```

| Attribute | Required | Description |
|---|---|---|
| `state` | Yes | API name of a lifecycle state in `{lifecycle_name}.{state_name}` dot-notation. Both the lifecycle and the state must already exist on the object. |

Multiple `vault:sectionState` elements stack as an OR condition: the section is visible when the
record is in any of the listed states.

---

## Section content types

### Detail form: `vault:detailform`

Displays fields in a one- or two-column grid.

```xml
<vault:detailform type="Two-Columns">
    <vault:field reference="name__v" />
    <vault:field reference="status__v" />
</vault:detailform>
```

| Attribute | Required | Allowed values | Description |
|---|---|---|---|
| `type` | Yes | `"One-Column"`, `"Two-Columns"` | Controls the column count. Two-column layouts fill left-to-right in declaration order. |

Fields inside a `vault:detailform` use `vault:field` with a `reference` attribute naming the
field's API name directly (no relationship prefix):

```xml
<vault:field reference="application__v" />
<vault:field reference="state_stage_id__sys" />
```

`width` is not valid on `vault:field` inside a `vault:detailform`; it is strictly a
`vault:relatedObject` attribute.

---

### Related object section: `vault:relatedObject`

Displays a table of records related to the current record via a named relationship.

```xml
<vault:relatedObject
    relationship="submission_countries__rimr"
    preventRecordCreate="false"
    modalCreateRecord="false"
    displayAsSimple="false">
    <vault:field reference="name__v" width="182px" />
    <vault:field reference="country__rim" width="182px" />
    <vault:field reference="country__rimr.region__v" width="182px" />
</vault:relatedObject>
```

| Attribute | Required | Description |
|---|---|---|
| `relationship` | Yes | API name of the relationship. The relationship name includes its namespace suffix (e.g. `__vr`, `__rimr`, `__cr`). |
| `preventRecordCreate` | No | `"true"` hides the inline record-create action. `"false"` allows it. |
| `modalCreateRecord` | No | `"true"` opens the create action in a modal dialog. `"false"` opens a full page. |
| `displayAsSimple` | No | Only meaningful for complex join sections (see below). `"true"` renders a complex join as a simple (pass-through) join, hiding the join object's own fields. `"false"` renders the join object's fields normally. Has no effect on referring or simple join sections. |

**Relationship types and `displayAsSimple`**

A `vault:relatedObject` section renders differently depending on the relationship type it
references. There are two top-level types:

- **Referring object**: The related object has a reference field pointing back to the current
  record's object. The section displays the referring records directly. `displayAsSimple` has no
  effect here.

- **Join object**: A join object sits between two parent objects, holding a reference field to
  each. This is how Vault models many-to-many relationships. Join object sections come in two
  further variants:

  - **Simple join (pass-through)**: The join object has no meaningful fields beyond its two
    parent reference fields. The section "passes through" to the other parent, showing its
    records as if the join object were invisible. This is the behavior `displayAsSimple="true"`
    also produces.

  - **Complex join**: The join object has additional fields of its own. By default the section
    exposes those fields. When the extra fields are not relevant to users, setting
    `displayAsSimple="true"` renders the complex join section in pass-through style, showing
    only the other parent's records.

Whether a relationship is a referring or join type is determined by how the relationship was
defined on the object, not by its namespace suffix (`__vr`, `__cr`, `__rimr`). Check the object's
Relationships tab in the Admin UI or retrieve the relationship metadata before deciding whether
`displayAsSimple` is applicable.

**Field references inside `vault:relatedObject`**

Each `vault:field` column in a related object section takes a `reference` and an optional `width`.
The `reference` attribute supports two forms:

| Form | Example | When to use |
|---|---|---|
| Direct field on the related object | `reference="name__v"` | A field that belongs to the related object's own record. |
| Traversed field (one hop) | `reference="country__rimr.region__v"` | A field on an object reached by following a reference field on the related record. Format: `{relationship_field_name}.{target_field_name}`. Maximum one hop; deeper traversal is not supported. |

`width` takes a CSS length string (e.g. `"182px"`, `"200px"`). ~ Other CSS unit types are unverified;
pixel values are the only form seen in examples.

**VQL filter criteria**

A `<vault:vqlFilterCriteria>` child on `vault:relatedObject` filters which related records are
shown, and can apply the same filter when the create action opens. Place it before the
`vault:field` children:

```xml
<vault:relatedObject relationship="my_relationship__cr" ...>
    <vault:vqlFilterCriteria query="status__v = 'active__v'" applyOnCreate="true" />
    <vault:field reference="name__v" width="200px" />
</vault:relatedObject>
```

| Attribute | Description |
|---|---|
| `query` | A VQL `WHERE`-clause fragment (no `SELECT`/`FROM`) that filters the related records shown. |
| `applyOnCreate` | `"true"` pre-applies the filter criteria when the user opens the inline create form. |

The child object and its reference field must exist before the parent layout can name them;
sequence layout statements last in a multi-statement MDL script.

---

### UI widget: `vault:uiwidget`

Renders an app-defined section built by an application team. These sections don't follow a
single standard pattern: each widget is specific to the app that authored it, with its own
rendering logic and configuration. This is a less common section type (roughly 5-10% of
sections in practice).

```xml
<vault:uiwidget name="submission_administrative_info_widget__v" />
```

| Attribute | Description |
|---|---|
| `name` | API name of the widget to render. The valid names are determined by the application that owns the widget. |

Do not author a `vault:uiwidget` section from scratch: the widget must already be deployed by
the application team that owns it. Retrieve a working `RECREATE Pagelayout` from a Vault that
already uses the widget to see the exact element form and any additional attributes it may
require.

---

### Workflow timeline: `vault:wftimeline`

Renders the workflow timeline panel. The element itself is self-closing with no attributes.
Section-level configuration (help text and lifecycle state visibility) is carried by the parent
`vault:section`:

```xml
<vault:section title="Workflow Timeline" name="workflow_timeline__v" helpContent="Here is some help.">
    <vault:sectionState state="submission_lifecycle_lifecycle__c.distributed_state__c" />
    <vault:sectionState state="submission_lifecycle_lifecycle__c.in_progress_state__c" />
    <vault:wftimeline />
</vault:section>
```

---

### Attachments: `vault:attachments`

Renders the native attachments panel. The element itself is self-closing with no attributes.
Section-level configuration (help text and lifecycle state visibility) is carried by the parent
`vault:section`:

```xml
<vault:section title="Attachments" name="attachments__v" helpContent="Attach files here.">
    <vault:sectionState state="submission_lifecycle_lifecycle__c.ha_received_state__c" />
    <vault:sectionState state="submission_lifecycle_lifecycle__c.in_progress_state__c" />
    <vault:attachments />
</vault:section>
```

---

## Authoring and sequencing notes

- Retrieve a working `RECREATE Pagelayout` from the target Vault (`GET /api/mdl/components/Pagelayout.{name}`)
  before writing markup from scratch: the response shows the current markup and the surrounding
  MDL attributes (`label`, `active`, `object`, `display_lifecycle_stages`, etc.) that must be
  restated in full on a `RECREATE`.
- Because the entire `page_markup` is replaced as one value, include every section in each
  `RECREATE` statement, even sections not being changed.
- Related object sections must be sequenced after the child object and its reference field have
  been created. A layout statement that references a relationship not yet deployed fails with a
  missing-dependency error.
- Relationship names in `vault:relatedObject` use the relationship's API name, including its
  namespace suffix (`__vr`, `__cr`, `__rimr`), not the related object's name.
