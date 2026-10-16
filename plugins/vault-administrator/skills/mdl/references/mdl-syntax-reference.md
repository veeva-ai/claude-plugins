# MDL Statement and Syntax Reference

This file keeps the basics: verify an optional attribute or a
component type it doesn't cover against `mcp__vault-docs-mcp__search_documentation` before writing
the statement.

## Command syntax

Every MDL command follows the same shape:

```text
COMMAND Componenttypename component_name (
  attribute_name(attribute_value),
  attribute_name(attribute_value),
  Subcomponenttypename subcomponent_name (
    attribute_name(attribute_value)
  )
);
```

- **Component type name**: capitalized first letter, no spaces or special characters
  (`Picklist`, `Objectlifecycle`, `Doclifecycle`).
- **Component name**: snake_case, ends in a namespace (`color__c`). Dot-notation expresses a
  parent/child hierarchy for component types that nest that way (`my_object__c.my_type__c`).
- **Subcomponent type name / name**: only meaningful inside the parent component; a
  subcomponent cannot stand alone. Subcomponent names can also use dot-notation
  (`my_object__c.my_field__c`).
- **Attribute name**: lowercase snake_case (`label`, `max_length`).
- **Attribute value**: shape depends on the attribute's data type (see below).

Statements end with a semicolon. Multiple statements in one script execute sequentially, but
**MDL scripts are not transactional**: a later statement failing does not roll back an earlier
one that already succeeded.

## The five commands

### CREATE

Fails if a component with the same name already exists.

```text
CREATE Picklist color__c(
   label('Color'),
   active(true),
   can_add_values(true),
   can_reorder_values(true),
   Picklistentry red__c(
      value('Red'),
      active(true),
      order(1)
   ),
   Picklistentry blue__c(
      value('Blue'),
      active(true),
      order(2)
   )
);
```

### RECREATE

An upsert: creates the component if it doesn't exist, otherwise replaces it wholesale to match
the statement. Anything live and not restated is dropped. Vault-generated VPKs always export
`RECREATE` statements; `GET /api/mdl/components/{Type}.{name}` also returns one. Same shape as
`CREATE`, command keyword swapped.

### RENAME

```text
RENAME Componenttypename old_component_name TO new_component_name;
```

Only works on custom (`__c`) components. Fails if a component already exists under the target
name.

### DROP

```text
DROP Componenttypename component_name;
```

Deletes the component and any subcomponent records under it. No documented undo path.

### ALTER

```text
ALTER Componenttypename component_name(
  attribute_name(attribute_value),
  ADD Subcomponenttypename subcomponent_name(
    attribute_name(attribute_value)
  ),
  MODIFY Subcomponenttypename subcomponent_name(
    attribute_name(attribute_value)
  ),
  RENAME Subcomponenttypename old_subcomponent_name TO new_subcomponent_name,
  DROP Subcomponenttypename subcomponent_name
);
```

Changes named attributes and/or subcomponents in place; everything else on the component is
untouched. `ADD`/`MODIFY`/`RENAME`/`DROP` subcommands can combine in one `ALTER`, each acting on
a different subcomponent. A direct attribute value (no subcommand keyword) simply overwrites
that attribute.

**Multi-value attributes** (attributes that hold a list, e.g. a `Docfieldlayout`'s `fields`)
take `ADD (...)`/`DROP (...)` forms to append or remove individual values without restating the
whole list:

```text
ALTER Docfieldlayout general__c (
  fields ADD ('Docfield.my_field__c')
);
```

Appended values land at the end of the list; if `ordered` is `true` on that attribute's
metadata, list order affects the component's behavior.

## Parser rules that don't show up in the command syntax

A few rules that hold in practice, beyond the reference pages:

- Empty parentheses are a value, not a placeholder. An attribute stated with no value inside an
  `ALTER` or `RECREATE` clears it, while an attribute omitted entirely is left unchanged.
  Retrieved `RECREATE` blocks routinely come back carrying `description()` or
  `available_lifecycles()`, so resubmitting one unedited wipes those attributes in the target.
- MDL has no comment syntax. A script can't carry a `//` or `#` explanatory line; keep notes
  outside the file that gets submitted.
- String literals can't sit next to each other to be concatenated. Build the full string as one
  quoted literal rather than writing two adjacent quoted pieces and expecting Vault to join them.
- You can't declare a standard (`__v`/`__sys`) subcomponent through MDL. Standard subcomponents
  are Veeva-owned; a script can reference one but not create or redefine it, consistent with the
  broader standard-namespace restriction under Naming and namespace conventions above.
- For a multi-type object, a field's membership in a given `Objecttype` is expressed through that
  type's `Typefield` subcomponent, which carries its own `source` attribute (`standard`, `sample`,
  `custom`, `application`, or `system`). ~ The docs describe `source` on `Typefield` as "the owner
  of the type," which reads like it's really about the type rather than the field-to-type link;
  treat the exact membership semantics of `source()` on a `Typefield` `ADD` as unconfirmed and
  check it against a retrieved component (Step 2) or a sandbox before relying on it.

## Operators

| Operator | Syntax | Behavior |
|---|---|---|
| `IF EXISTS` | `ALTER Componenttypename IF EXISTS component_name (...)` | Executes only if the component already exists; otherwise a no-op. |
| `IF NOT EXISTS` | `RECREATE Componenttypename IF NOT EXISTS component_name (...)` | Executes only if the component does not already exist; otherwise a no-op. |

Useful for a script meant to run unchanged across multiple Vaults regardless of each Vault's
starting state. A guarded statement returning success tells you nothing about which branch it
took; check the component afterward if that distinction matters.

## Attribute data types and quoting

| Type | Validation | Example |
|---|---|---|
| String | Single-quoted (`'`), never double-quoted. Max length per attribute metadata. | `label('My Label')` |
| Boolean | `true`/`false`, unquoted. | `active(true)` |
| Number | Unquoted; bounded by `max_value`/`min_value`/`scale` on the attribute metadata. | `max_value(2147483647)` |
| Component | A reference to another component, still expressed as a quoted string in `{Type}.{name}` form. | `object('Doctype.my_special_doctype__c')` |
| Enum | One value from a fixed allowed list, quoted like a String. | `content_type('Document')` |
| XMLString | Wrapped in curly braces `{}`. Validated for XML well-formedness only, not schema correctness; malformed XML returns `GEN-F***-II-1056`. | `attribute({<xml>...</xml>})` |
| LongString | Same quoting as String, for Long Text-sized values. | `attribute('...')` |
| SdkCode | Java SDK source, wrapped in a `<VeevaData>` node. | `source_code(<VeevaData>...</VeevaData>)` |
| Expression | A formula, wrapped in square brackets. | `formula([field__c='abc'])` |
| File | A reference to a staged binary file by its returned name (see Content-Bearing Components below). | `template('4be398c32fc2ccf48adaf6ebe53782a1')` |

Component references don't all take the same form. On an object `Field`, `object` takes the bare
object name (`object('project__c')`) while `picklist` takes the type-qualified name
(`picklist('Picklist.item_type__c')`); qualifying the object name or bare-naming the picklist
fails validation. Check the attribute's own metadata rather than assuming one convention holds
across a statement.

The single-quoted-string rule is the one most worth internalizing: it applies uniformly across
`CREATE`/`RECREATE`/`ALTER`, and a double-quoted string is one of the more common copy-paste
authoring mistakes (e.g. porting a statement from a source that used double quotes for JSON).
Vault rejects it at parse time with `GEN-F-402` "no viable alternative at input". A single quote
inside a string is escaped with a backslash: `('hire_date__c >= \'2026-10-01\'')`.

## Naming and namespace conventions

- Component/subcomponent names are lower-case, snake_case, and end in a double-underscore
  namespace: `__c` (custom), `__v` (Veeva standard), `__sys` (system).
- Names must be 40 characters or fewer, excluding the namespace suffix.
- You cannot delete a component with a standard namespace (`__v`/`__sys`) and can generally only
  make limited changes to one (e.g. updating a label): most of the surface is Veeva-owned.
- Code component names (`Recordtrigger`, `Documentaction`, `Recordaction`, `Userdefinedclass`)
  are the fully-qualified Java class name instead of a snake_case token, e.g.
  `Recordtrigger.com.veeva.vault.custom.triggers.HelloWorld`.

## Content-bearing components

`Formattedoutput`, `Overlaytemplate`, and `Signaturepage` hold binary content that does not
travel through `/api/mdl/execute` directly: the MDL statement only references a staged file by
name.

**Upload**: `POST /api/mdl/files`, `multipart/form-data`:

```json
{
  "responseStatus": "SUCCESS",
  "data": {
    "name__v": "4be398c32fc2ccf48adaf6ebe53782a1",
    "format__v": "application/pdf",
    "size__v": 4716,
    "sha1_checksum__v": "4be398c32fc2ccf48adaf6ebe53782a1"
  }
}
```

Reference the returned `name__v` in the component's file-typed attribute:

```text
RECREATE Formattedoutput my_formatted_output__c (
    label('My Formatted Output'),
    active(true),
    root_object('Object.product__v'),
    root_object_type('Objecttype.product__v.base__v'),
    output_type('native'),
    template('4be398c32fc2ccf48adaf6ebe53782a1')
);
```

Once a staged file is referenced by a component, Vault stops serving it from the staging area.
**Retrieve** the deployed file afterward with `GET /api/mdl/components/{Type}.{name}/files`,
whose response includes the same `sha1_checksum__v` alongside `original_name__v` and a download
link; compare it against a local hash to confirm integrity end to end. To change a component's
file, upload the new one first, then update the component to reference the new name.

This `sha1_checksum__v` is unrelated to the `.md5` file that accompanies an `.mdl` step file
inside a hand-assembled VPK (a package-integrity check on the script text, generated locally via
`certutil -hashfile`/`md5`). Different mechanism, different layer; it belongs to package migration, not MDL authoring.

## Execute MDL Script endpoints

Both take raw MDL text as the request body; the body must begin with `CREATE`, `RECREATE`,
`RENAME`, `ALTER`, or `DROP`. Neither endpoint takes a `{version}` path segment.

**Synchronous**: `POST /api/mdl/execute`

```json
{
    "responseStatus": "SUCCESS",
    "script_execution": {
        "code": "GEN-S-0",
        "message": "OK",
        "warnings": 0,
        "failures": 0,
        "exceptions": 0,
        "components_affected": 1,
        "execution_time": 0.028
    },
    "statement_execution": [
        {
            "vault": "myvault.veevavault.com",
            "statement": 1,
            "command": "RECREATE",
            "component": "Picklist.color__c",
            "message": "[SUCCESS] RECREATE Picklist color__c",
            "response": "SUCCESS"
        }
    ]
}
```

**Asynchronous**: `POST /api/mdl/execute_async`, same body, returns immediately:

```json
{
   "responseStatus": "SUCCESS",
   "script_execution": { "code": "GEN-S-0", "message": "OK", "warnings": 0, "failures": 0, "exceptions": 0, "components_affected": 1, "execution_time": 0.622 },
   "job_id": 327103,
   "url": "/api/v20.3/services/jobs/327103"
}
```

Retrieve results once the job completes: `GET /api/mdl/execute_async/{job_id}/results`, same
`script_execution`/`statement_execution` shape as the sync response.

**Required** (not just available) for a script touching 10,000+ records of a `raw` object doing
any of: enabling lifecycles; enabling or disabling object types; adding or removing a field;
changing the max length of a variable-length field (Text/Long Text/Rich Text); adding or
removing an `Index`; or changing the fields that compose an `Index`. While queued or running,
the object's `configuration_state` is `IN_DEPLOYMENT` and its fields/layouts/indexes can't be
edited; only one such deployment can be queued at a time. Cancel a not-yet-started deployment
with `POST /api/{version}/metadata/vobjects/{object_name}/actions/canceldeployment` (this one
endpoint is versioned, unlike its MDL siblings).

Read `responseStatus` and `script_execution.failures`/`.exceptions` rather than trusting HTTP
200 alone: a script can execute with an overall 200 while individual `statement_execution`
entries report `FAILURE`.

## Error reference

- **`GEN-F***-II-1006`-style, "Required attribute [X] is missing"**: attribute set didn't match
  what the validator demands for that operation; check `context` in the error body.
- **`GEN-F***-II-1056`**: malformed XMLString (well-formedness only, not schema correctness).
- Missing-dependency failures surface as a reference to a component that doesn't exist yet in
  the target Vault; order statements so dependencies land first.

~ A canonical, exhaustive MDL error-code table was not found in the current documentation pull;
treat any error code not covered above as needing a direct MCP or sandbox check rather than a
guess at its meaning.
