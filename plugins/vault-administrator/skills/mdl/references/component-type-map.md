# MDL Component Type Quick Map

A starting point for common admin-facing component types, not the full catalog. For a type's
attributes and Supported Operations table, look it up with
`mcp__vault-docs-mcp__search_documentation` against the `vault_developer_documentation` source
first; `general.veevavault.dev/mdl/component-types` (several hundred types) is the fallback when
the MCP doesn't have it.

Every type has its own **Supported Operations** table (CREATE/RECREATE/ALTER/DROP/RENAME/
Describe/Generate RECREATE/Queryable) on its own doc page: check the specific type before
writing a statement against it rather than assuming from a similar-sounding type. The patterns
below repeat often enough to be worth knowing up front.

Doc pages and this table both go stale against what a given Vault actually has deployed. Before
trusting either for an attribute set, pull a real component of that type and read the attributes
off of it:

```
GET /api/mdl/components/Picklist.study_phase__c
```

The response is a plain-text `RECREATE Picklist study_phase__c (...)` block with every attribute
Vault populated for that component, which beats guessing at an optional or type-dependent
attribute from the doc page alone.

## Reading the Supported Operations table

- **"Yes\*"** on CREATE/RECREATE/ALTER for a metadata-class type almost always carries a note:
  *"MDL Operation is supported, but the Component Type contains field(s) that use proprietary
  XML. Direct modification of the XML is not supported or recommended. Instead, creating and/or
  updating the component is better suited through the Admin UI."* This shows up on
  `Doclifecycle`, `Objectlifecycle`, `Objectworkflow`, `Dashboard`, and most workflow/lifecycle
  types. MDL will technically accept a script, but the XML-bearing attributes (`entry_action`,
  `security_settings`, `dashboard_markup`, etc.) are hand-authored risk, not a place to
  freehand-generate content.
- **"ALTER: may not be supported for some multi-value attributes"** is a common qualifier: don't
  assume every multi-value attribute on a type ALTERs cleanly; verify by re-retrieving the
  component after the change (SKILL.md Step 7's silent-no-op failure mode).
- **Queryable = Yes** means the component type surfaces through VQL against `vault_component__v`
  (or, for code components like `Recordtrigger`/`Recordroletrigger`, is queryable but has every
  MDL command operation marked unsupported on its own page: read as "manage this type's Java
  source through the SDK/VPK code-deployment path, not raw MDL," despite the Execute MDL Script
  API reference showing a worked `RECREATE Recordtrigger` example. ~ This is a documented
  contradiction; verify against a sandbox before relying on either page alone).
- A blank or "No" entry for an operation on one page has, in practice, sometimes still worked via
  `POST /api/mdl/execute` for other component types (e.g. `Objectaction`, per Vault Java SDK
  deployment tooling). Treat a "No" as "not guaranteed, test it," not as a hard wall, but don't
  build a workflow around an undocumented capability without a sandbox check first.

## Metadata components (configuration elements)

| Component Type | Typical use | Notable attributes | Ops notes |
|---|---|---|---|
| `Object` | Custom or standard Vault object definition. | `label`, `label_plural`, `active(true)`, `in_menu(false)`, `allow_attachments(false)`, `object_class('base')`, `audit(true)`, `dynamic_security(false)`, `system_managed(false)` (always false for custom objects), nested `Field` subcomponents. In a `CREATE` or `RECREATE`, declare only custom (`__c`) `Field` subcomponents; `name__v` is the one standard field you configure inline (label, `max_length`). Vault creates every other standard field itself and rejects one that is declared with `GEN-FADDS-ONA-1019` (`[standard] sub-components cannot be created`). | Describe is only via `/api/{version}/metadata/vobjects`, not the general configuration endpoint. |
| `Field` (subcomponent of `Object`) | A field on an object. | `type`, `active(true)`, `required`, `unique(false)`, `list_column(false)`, `order` (0-based position among custom fields), `max_length`/`max_value`/`min_value`/`scale` (type-dependent), `picklist`. `type` is one of `String`, `Number`, `Boolean`, `Date`, `DateTime`, `Object`, `Picklist`, `ID`, `users`, `Component`, `documents`, `LongText`, `RichText`, `Binary`, `Attachment`, `ObjectReference`, `ObjectParent`. | `ADD Field` via `ALTER Object` demands a stricter attribute set than a retrieved `RECREATE` shows; see SKILL.md Step 7. |
| `Objecttype` / `Typefield` | Object type definitions and each type's field membership. | Type: `label`, `active`, nested `Typefield` subcomponents. `Typefield`: `source`, `required`, `default_value`, `pickentries`, `relationship_criteria`. | A field added by `ALTER Object ... ADD Field` applies to the base type only: every non-base type needs `ALTER Objecttype {object}.{type} (ADD Typefield {field}(source('custom'), required(false)));` or the field is absent from those records and a later load silently drops the values. `Typefield` also carries per-type overrides beyond membership: `pickentries` restricts which picklist values that type accepts, `required` and `default_value` override the object-level setting for that type only, and `relationship_criteria` constrains the referenced object by VQL. A type-level default cannot override an object-level default back to blank. |
| `Pagelayout` | Object record detail page layout. | The whole layout lives in the single `page_markup` XMLString attribute, max 64,000 characters. | No section-level subcomponent exists. Both `RECREATE` and `ALTER` are supported (ALTER = Yes*). Either way, the full XML must be supplied for `page_markup`: retrieve the current markup, edit it, and resubmit. Use `ALTER` when only `page_markup` or another scalar attribute is changing; use `RECREATE` when replacing the full component definition. DROP = No, RENAME = Yes. A related-records section references the relationship name rather than the object (`<vault:relatedObject relationship="projects__cr">`), and `<vault:vqlFilterCriteria query="..." applyOnCreate="true"/>` filters it, so the child object and its reference field must exist before the parent's layout can name them: sequence layout statements last. |
| `Layoutrule` | Conditional show/hide rule on a page layout. | `label`, `active`, `expression` (required); `order`; multi-value `hide_sections`, `hide_fields`, `hidden_pages`. | A dot-notation child of its layout that can be created independently of it: `CREATE Layoutrule {layout_name}.{rule_name} (label('...'), active(true), order(100), expression('migrated_record__c = false'), hide_sections('migrated_legacy_data__c'));` |
| `Picklist` / `Picklistentry` | Picklist definitions and their values. | `label`, `active`, `can_add_values`, `can_reorder_values`; entries take `value`, `order`, `active`. | Full CREATE/RECREATE/ALTER/DROP/RENAME support, no XML caveat: one of the more MDL-friendly types. |
| `Doctype` | Document type taxonomy. | `label`, `fields` (linked `Docfield`s), `available_lifecycles`, `document_number_format`. | ALTER caveated for multi-value attributes; no XML caveat. |
| `Docfield` | A document field, standard or custom. | `type`, `max_length`, `picklist`, `required`, `shared`/`local` via `defined_in`/`used_in`. | Same shape as an object `Field`, different component type. |
| `Doclifecycle` / `Doclifecyclestate` / `Doclifecyclerole` | Document lifecycle states, roles, and rules. | XML-typed `entry_criteria`, `entry_action`, `user_action`, `security_settings`. | XML caveat applies; DROP/RENAME unsupported on `Doclifecycle` itself. |
| `Objectlifecycle` / `Objectlifecyclestate` | Object record lifecycle states and rules. | Same XML-rule shape as `Doclifecycle`, object-side. | XML caveat applies; DROP/RENAME supported (unlike `Doclifecycle`). |
| `Objectlifecyclestateuseraction` | A user action offered in one state of an object lifecycle. | Named `{state}.{action}`. `label`, `order`, and `rule(@rule.xml)`, where the rule is a sidecar file holding `<rule><actions><action .../></actions></rule>`. | Two action types cover most of it: `type="LIFECYCLE_STATE_CHANGE_ACTION" transitionState="Objectlifecyclestate.{state}"` moves the record, and `type="LIFECYCLE_RUN_WORKFLOW_ACTION" objectWorkflow="Objectworkflow.{workflow}"` starts a workflow. Copy a working one from the workspace and change the target. A workflow action needs an active workflow on the same lifecycle, one action per state for that workflow, and a Workflow Cancel State set on every state the workflow can start from (Vault Help, Configuring Object Workflows); when the example is the same workflow started from another state, only the state and the label change. The Vault docs do not describe this XML; the shape comes from a pulled example of the same action type in the workspace, which is why the skill copies an example rather than typing it. |
| `Objectworkflow` / `Objectworkflowstep` | Object workflow definition and steps. | `object_lifecycles`, `workflow_content_type`, XML-typed `step_detail`. | XML caveat applies. |
| `Docrelationshiptype` | Document-to-document relationship type. | `target_doctype`, `single_use`, XML `relationship_filter`. | Full CREATE/RECREATE/ALTER/DROP/RENAME. |
| `Report` / `Reporttype` | Vault reports and their underlying report types. | XML `definition`/`layout`/`configuration`, `content_type`. | `Report`: ALTER unsupported (CREATE/RECREATE/DROP/RENAME only), and `active` is not editable, so there is no inactivating a report. A report a `Reporttype` references can't be dropped (`RPT-FDRO-II-33023`) until that reference is gone. XML caveat applies. |
| `Dashboard` | Vault dashboards. | XML `dashboard_markup`, `editors`/`viewers`/`owners`. | XML caveat applies. |
| `Tab` / `Subtab` | Top-level tabs and the subtabs under them. | `label`, `order`, `object` for a record tab; `page` and `url` for one that hosts a Custom Page; `parent` on a subtab. | Add a subtab with `ALTER Tab {tab} ( ADD Subtab {name}(...) );`. A tab or subtab hosting a Custom Page needs `url('https://${Vault.domain}/ui/#custom/page/${Page.url_path_name}')` next to `page('Page.{name}')`; without it the statement fails with `PAG-FADDS-ONA-44016` "Attribute url is required". Put a statement that carries `${...}` in a file and run `vault mdl "$(cat tab.mdl)"`, or send the file with the curl form in SKILL.md. Inside double quotes bash stops on `${Vault.domain}` as a bad substitution and PowerShell swaps it for an empty string, and the statement's own single quotes rule out a single-quoted command line. |
| `Objectaction` | Registers a deployed `RecordAction` (Java SDK) as a user action on an object. | `label`, `action('Recordaction.<FQCN>')`, `available_all_states`. | Dev-portal ops table is commonly blank/misleading for this type; RECREATE empirically works, see SKILL.md Step 7. `available_all_states(true)` is required on objects without an `Objectlifecycle`, or the action silently fails to appear. |
| `Formattedoutput` / `Overlaytemplate` / `Signaturepage` | Content-bearing templates. | `template`/file-typed attributes referencing a staged upload. | Go through `/api/mdl/files` first; see `mdl-syntax-reference.md`. |
| `Atomicsecurity` | Field-level security rules by object lifecycle state and role. | `object`, `object_lifecycle`, `state`, nested `Fieldsecurity`. | Used for bulk/programmatic security config across many objects/states: the case MDL is built for over the Admin UI. |
| `Tagsecurityrule` / `Appsecurityrule` | System-defined security rule types; external admins can only attach `*assignment` subcomponents. | `object` reference on the assignment subcomponent. | CREATE unsupported on the parent type; RECREATE/ALTER supported only for the subcomponents. |
| `Matchingrule` | Dynamic Access Control matching/sharing rule. | `object`, `role`, `user_role_fields`, `data_fields`. | Full support, no XML caveat. |
| `Notificationtemplate` | Email/notification templates. | `subject`, `email_body`, `entity_type`. | DROP supported for custom components only, not system ones. |

## Code components (Java SDK-backed)

| Component Type | Typical use | Notes |
|---|---|---|
| `Recordtrigger` | Object record trigger Java class. | Component name is the fully-qualified class name. `source_code` is `SdkCode` type (`<VeevaData>` wrapper). Its own component-type page shows every MDL operation as unsupported, but the Execute MDL Script API reference shows a working `RECREATE Recordtrigger` example. ~ Contradictory across docs; confirm behavior against a sandbox rather than trusting either page alone. |
| `Recordroletrigger` | Trigger on manual record-role assignment/removal. | Same shape and same documented contradiction as `Recordtrigger`. |
| `Documentaction` / `Recordaction` / `Userdefinedclass` | Other Java SDK code component types. | ~ Not independently verified in this pass; expect the same FQCN-as-name pattern as `Recordtrigger`, confirm specifics via `GET /api/mdl/components/{Type}.{FQCN}` against a known-deployed example before authoring. |

For any code component, writing the Java source itself is out of scope for this skill, whatever
form the request takes: an ungrounded body of SDK source would read as authoritative when it
isn't, and that holds just as much for a "draft it anyway" or "outside the skill" framing as for a
direct ask. Point at the vault-developer skill that owns the code's job (the triggers skill for a
`Recordtrigger`, and its siblings for the other code types) and stop at the MDL statement that
registers or updates the component record referencing already-deployed code.

## When the type isn't in this table

1. Retrieve a working example from the target Vault: `GET /api/mdl/components/{Type}.{name}`
   (see SKILL.md Step 2), the fastest way to see the real attribute set and confirm which
   commands actually work, regardless of what the doc page's table claims.
2. Fall back to `mcp__vault-docs-mcp__search_documentation` against `vault_developer_documentation`
   for `{type name} component type MDL` to pull the specific attribute table and Supported
   Operations block before writing anything.
