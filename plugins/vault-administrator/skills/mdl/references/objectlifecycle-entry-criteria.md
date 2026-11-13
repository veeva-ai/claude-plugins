# Object Lifecycle Entry Criteria: the `rule` XML

Read this before writing or changing an `Objectlifecyclestateentrycriterion`. Its `rule`
attribute is an XMLString, and Vault's component reference marks this XML as proprietary:
"Direct modification of the XML is not supported or recommended," with the Admin UI named as
the better path. Say that to the user once, up front. MDL is still the right tool when the same
criterion has to land on several Vaults or be reviewed as text, and in that case run it on a
sandbox first and read the component back.

The safest starting point is a criterion that already exists. Retrieve the lifecycle (SKILL.md
Step 2), find a criterion of the kind being asked for, and adapt it. Write one from the rules
below only when there is nothing to copy.

## Statement shape

```
ALTER Objectlifecycle <lifecycle_name> (
    ADD Objectlifecyclestateentrycriterion <state_name>.<criterion_name>(
        rule({<xml>}),
        order(<n>)
    )
);
```

- `ADD` sits directly inside the `ALTER Objectlifecycle (...)` body. Do not wrap it in
  `MODIFY Objectlifecyclestate <state> (...)`: an entry criterion is a sibling of the state, not
  its child. The dot in `<state_name>.<criterion_name>` is what ties it to the state.
- `rule` is required. `order` is a number.

## Adding a criterion, or adding a condition to one

Look at the criteria already on the target state before doing either. The name, the `order()`
value, and the XML to reproduce all depend on what is there.

- **A new criterion on a state:** `ADD Objectlifecyclestateentrycriterion <state>.<new_name>(...)`.
  Name it `<state>.entrycriteria<N>__c`, where N is one more than the number of criteria already
  on that state. The first criterion on a state takes `order(1000)`, and each new one adds 1000.
- **Another condition on an existing criterion:**
  `MODIFY Objectlifecyclestateentrycriterion <state>.<existing_name>(...)`, same name, `order()`
  left alone. `rule` has no append syntax, so reproduce the whole existing
  `<rule><actions>...</actions></rule>` with the new `<action>` added beside the old ones, and
  resubmit it whole.

## The `rule` document

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rule>
  <actions>
    <action type="LIFECYCLE_ENTRY_CRITERIA_FIELD">
      <expression operator="isNotBlank">
        <expression operator="field" type="date"><value>Object.milestone__v.actual_finish_date__v</value></expression>
      </expression>
      <errorMessage>Actual Finish Date is required before this record can move forward.</errorMessage>
    </action>
  </actions>
</rule>
```

- `<rule>` holds exactly one `<actions>`, and optionally one `<expression>` before it that gates
  whether the rule applies at all. Anything else is rejected.
- Each `<action>` is one check. Give every action an `<errorMessage>` the user will understand.
- `action type="LIFECYCLE_ENTRY_CRITERIA_FIELD"` is the field check, covered first. The other
  action types, formulas and related records and documents, have their own XML shapes and are
  covered after it. Read "Action types" to pick one.

## Field references

Always three dot-separated parts, starting with the literal `Object`:

```
Object.<object_name>.<field_name>
```

`Object.quality_event__qdm.date_closed__qdm`, never the bare field name.

## Choosing the operator

Pick it in this order: what the requirement means, then whether the field's data type allows
it. Vault enforces the table below on the server. It is not a UI suggestion.

| Field type | Operators allowed |
|---|---|
| Object reference | `eq`, `ne`, `in`, `isBlank`, `isNotBlank` |
| Number | `eq`, `ne`, `gt`, `lt`, `isBlank`, `isNotBlank` |
| Text | `eq`, `ne`, `startsWith`, `isBlank`, `isNotBlank` |
| Yes/No (Boolean) | `eq`, `isBlank`, `isNotBlank` |
| Long Text | `isBlank`, `isNotBlank` |
| Picklist | `eq`, `ne`, `in`, `isBlank`, `isNotBlank` |
| Date, DateTime | `lastDays` `lastWeeks` `lastMonths` `lastYears`, `nextDays` `nextWeeks` `nextMonths` `nextYears`, `notLastDays` `notLastWeeks` `notLastMonths` `notLastYears`, `isBlank`, `isNotBlank` |

How many children each operator takes:

| Operator | Children |
|---|---|
| `isBlank`, `isNotBlank` | 1: the field. No value. |
| `gt`, `lt`, `gte`, `lte`, `startsWith`, and the relative-date operators | 2: the field, then the value |
| `between` | 3 |
| `eq`, `ne`, `in`, `notIn` | 2 or more: the field, then one or more values |

Never use `contains`, `contains_ci`, or `search` here, even when the requirement says
"includes" or "mentions". They parse and do nothing in an entry criterion. `startsWith` is the
only text match available. There is no `and` or `or`: a second condition that must also hold is a second `<action>`.

## Negation

"Must not be X" does not mean `ne`, and it never means combining expressions.

- If the field type's row above has `ne`, use `ne`.
- If it does not, as on a Yes/No field, use `eq` and flip the value. "Must not be Yes" on a
  Yes/No field is `eq` against `false`. Tell the user this may also stop a record whose field is
  blank, and ask whether blank should be let through.
- Long Text has no `ne` and no `eq` either, only `isBlank` and `isNotBlank`.

"Must not be Yes" on a Yes/No field:

```xml
<action type="LIFECYCLE_ENTRY_CRITERIA_FIELD">
  <expression operator="eq">
    <expression operator="field" type="boolean"><value>Object.product_complaint__c.death_or_serious_injury_involved__c</value></expression>
    <expression operator="value" type="boolean"><value>false</value></expression>
  </expression>
  <errorMessage>Death or Serious Injury Involved must not be Yes before this record can be closed.</errorMessage>
</action>
```

The two wrong answers and what Vault says to each:

| Tried | Vault's error |
|---|---|
| `or(isBlank, eq false)` | `operator [Expression [op:OR]] is not supported` (`OLC-FADDS-II-1044`) |
| `ne` against `true` on a Yes/No field | `Can only use the following operators: [EQ,IS_BLANK,IS_NOT_BLANK]` (`OLC-FADDS-II-12517`) |

## Values and the `type` attribute

- `type` goes only on the leaves, `operator="field"` and `operator="value"`. On a comparison
  node (`eq`, `ne`, `isNotBlank`, ...) it is an error.
- `type` has to match what the value actually is: `string`, `number`, `boolean`, `date`. A
  non-numeric value marked `type="number"` fails.
- A picklist comparison uses the entry's **label**, not its API name:

```xml
<expression operator="eq">
  <expression operator="field" type="string"><value>Object.quality_event__qdm.reason_for_change__qdm</value></expression>
  <expression operator="value" type="string"><value>Change Control</value></expression>
</expression>
```

  Ask the user for the label, or read it from the picklist. Do not guess one from the API name.

## Action types

Five `type` values are documented here. The child shape under each is different, and putting the
wrong shape under a correct `type` is the most likely mistake.

| Requirement | `<action type="...">` | What goes inside |
|---|---|---|
| A field on this record | `LIFECYCLE_ENTRY_CRITERIA_FIELD` | comparison `<expression>` tree (sections above) |
| A formula that must be true | `objectLifecycleEntryCriteria.all.validateFormulaEvaluatesToTrue` | plain text formula in `<expression>` |
| Related records in a named state | `objectLifecycleEntryCriteria.all.validateRelatedRecordEntryCrit` | `function` expression tree |
| Related records in a state type | `objectLifecycleEntryCriteria.all.validateRelatedRecordStateType` | `function` expression tree |
| Related documents in a state type | `objectLifecycleEntryCriteria.Quality.stateTypeOfRelatedDocuments` | flat child elements, no `<expression>` |

Copy the `type` literal exactly. `validateRelatedRecordEntryCrit` is not
`validateRelatedRecordEntryCriteria`. Vault has other entry criteria types that are not in this
table. Do not write one from memory or by analogy: retrieve a lifecycle that already has one and
adapt it, or send the user to Admin > Object Lifecycles.

`<errorMessage>` is a child of `<action>` for every type. Vault adds `publicKey`, `privateKey`,
`actionPublicKey` and `actionPrivateKey` attributes when a criterion is read back. They are never
authored, so leave them out of an `ADD` or `MODIFY`.

## Formula is true

`type="objectLifecycleEntryCriteria.all.validateFormulaEvaluatesToTrue"`. The state change is
blocked when the formula is false.

- `<expression>` holds the formula as text. It has no `operator`, no `type`, and no children.
- The blob is XML, so escape the formula: `>` is `&gt;`, `<` is `&lt;`, `&` is `&amp;`. An
  unescaped `>` makes the whole `rule` malformed and the statement fails before Vault reads the
  formula.
- Fields are referenced by bare API name (`occurrence_count__c`), not the three-part
  `Object.<object>.<field>` form the field check needs. Do not mix them.

```xml
<action type="objectLifecycleEntryCriteria.all.validateFormulaEvaluatesToTrue">
  <expression>occurrence_count__c &gt; 1</expression>
  <errorMessage>There must be more than 1 occurrence count</errorMessage>
</action>
```

## Related records: pick the type first

A condition on a related record can be written two ways that both parse and both run. The
formula form (`complaint_owner__cr.status__v = 'Active'`) compares a field value. The related
record form compares a lifecycle state. When the related object has an object lifecycle, the
state form is the right one: the formula drifts when the field and the lifecycle disagree, and an
admin reading the lifecycle never sees it.

Deciding is a lookup. Retrieve the related object and read `available_lifecycles`:

```
GET /api/mdl/components/Object.<related_object_name>
```

| Related object's lifecycle | Requirement names | Use |
|---|---|---|
| present | a specific state ("Active", "Approved") | `validateRelatedRecordEntryCrit` |
| present | a state type ("any complete state") | `validateRelatedRecordStateType` |
| absent | a field value on the related record | `validateFormulaEvaluatesToTrue`, one hop `__cr` |

- **Which object to retrieve.** The one whose records get checked, which owns the relationship
  field. Outbound (the field is on the gated object): retrieve what its `object(...)` attribute
  points at. Inbound (the field is on the related object, pointing back): retrieve that related
  object.
- **Use the name verbatim.** `object('user__sys')` means `Object.user__sys`, not `Object.user`.
  A guessed name returns `MALFORMED_URL` or not-found, which reads like evidence and is not.
- **Do not switch endpoints after a failed retrieve.** Re-issue the same retrieve with the
  corrected name. The answer is `available_lifecycles`, never a status field in a field list.
- **A status field is not an answer.** `status__v` on `user__sys` looks like "active member", but
  `user__sys` has `vault_membership_lifecycle__sys` and that state is what the criterion checks.
  Status fields and lifecycles coexist.

## State of related record

`type="objectLifecycleEntryCriteria.all.validateRelatedRecordEntryCrit"`. Related records, reached
by an outbound or inbound relationship, must be in a named lifecycle state.

The shape is a function-call tree. Both related-record types share it:

```xml
<expression operator="function" qualifier="platform.all.validateRelatedRecord">
  <expression operator="method" qualifier="platform.all.validateRelatedRecord" type="string">
    <value>EQUALS</value>
  </expression>
  <expression operator="argumentGroup" type="string">
    <value>ARGUMENT_GROUP1</value>
    <expression operator="argument" type="string"><value>Object.contract__c.contract_owner__c</value></expression>
  </expression>
  <expression operator="argumentGroup" type="string">
    <value>ARGUMENT_GROUP2</value>
    <expression operator="argument" type="string"><value>Objectlifecycle.vault_membership_lifecycle__sys.active_state__sys</value></expression>
  </expression>
  <expression operator="argumentGroup" type="string">
    <value>ARGUMENT_GROUP3</value>
    <expression operator="argument" type="string"><value>OUTBOUND</value></expression>
  </expression>
</expression>
```

| Part | Holds |
|---|---|
| `qualifier` on `function` and on `method` | the same value on both. `platform.all.validateRelatedRecord` here. It is not the action `type` and the two are not interchangeable |
| `method` value | `EQUALS` for one record, `ALL_EQUALS` for many. Other values, not yet confirmed: `NONE_EQUALS`, `NOT_EQUALS`, `NOT_EMPTY`, `ALL_INCLUDES`, `NONE_INCLUDES`, `AT_LEAST_ONE_EQUALS`, `AT_LEAST_ONE_INCLUDES`, `INCLUDES`, `NOT_INCLUDE` |
| `ARGUMENT_GROUP1` | the relationship field, `Object.<object>.<field>` |
| `ARGUMENT_GROUP2` | the target state, three parts: `Objectlifecycle.<lifecycle>.<state>` |
| `ARGUMENT_GROUP3` | `INBOUND` or `OUTBOUND` |

- `type="string"` sits on `method`, `argumentGroup` and `argument`. `function` carries `qualifier`
  and no `type`.
- **Direction is structural, not a reading of the requirement.** `OUTBOUND`: the group 1 field
  lives on the gated object and points at one related record (`EQUALS` fits). `INBOUND`: the field
  lives on the related child object and points back, so there are many records and the method is
  usually an `ALL_`, `NONE_` or `AT_LEAST_ONE_` variant. Say which direction the criterion uses,
  and ask the user when the requirement doesn't say which relationship it means.
- `NOT_EMPTY` means "at least one related record exists" and needs no group 2.

### Narrowing with a filter

An action may carry a second top-level `<expression>`, a filter that limits which related records
the condition covers. It is the same function shape with `qualifier="platform.all.conditionFilterField"`
on both nodes, `method` value `EQ`, and its groups are numbered from 0:

| Group | Holds |
|---|---|
| `ARGUMENT_GROUP0` | `INBOUND` or `OUTBOUND` |
| `ARGUMENT_GROUP1` | the relationship field, same as the condition's group 1 |
| `ARGUMENT_GROUP2` | the field filtered on, e.g. `Object.<object>.object_type__v` |
| `ARGUMENT_GROUP3` | the filter value, format below |
| `ARGUMENT_GROUP4` | the value's datatype, lowercase, e.g. `string` |

Group 3 follows the datatype of the group 2 field, and the two known cases are opposites:

| Filtered field | Group 3 holds |
|---|---|
| Object type (`object_type__v`) | the object type's label, `Change Order` |
| Picklist (`action_type__c`) | the entry's API name, `preventive__c`, not `Preventive` |

A picklist label fails with `OLC-FADDS-II-1023` ("does not resolve to a valid [PicklistValue]"),
which means a label is sitting where an entry name is needed. Do not carry either form to another
datatype (reference, Yes/No, date). Retrieve a criterion that already filters on that datatype and
copy it. Group 4 for a picklist filter is not established, so read it back from a working
statement instead of assuming `string`.

A filter is the only way to narrow a related-record criterion. `or` is rejected with
`OLC-FADDS-II-1044` and `and` should be assumed the same. Two independent checks are two
sibling `<action>` elements.

## State type of related record

`type="objectLifecycleEntryCriteria.all.validateRelatedRecordStateType"`. Any, all or none of the
related records must be in a lifecycle state type rather than a named state. With `NOT_EMPTY` it
only asserts that a related record exists.

Same tree as above, with two differences:

- `qualifier` is `platform.all.validateRelatedRecordStateType` on `function` and `method`.
- `ARGUMENT_GROUP2` is two parts: `Lifecyclestatetype.<state_type>`, for example
  `Lifecyclestatetype.initial_state__sys` or `Lifecyclestatetype.complete_state__sys`.

The optional filter and the direction rules are the same as for State of related record.

## State type of related documents

`type="objectLifecycleEntryCriteria.Quality.stateTypeOfRelatedDocuments"`. Documents related
through a document relationship must be in a document lifecycle state type, for example every
related document steady before a change control record moves on.

The literal says `.Quality.`, not `.all.`. Whether it exists on a Vault outside the Quality
application family is unconfirmed, so say that and check the target Vault (retrieve a lifecycle
that uses it, or look in Admin > Object Lifecycles) before writing one for a non-Quality Vault.

This type has flat children and no `<expression>`:

```xml
<action type="objectLifecycleEntryCriteria.Quality.stateTypeOfRelatedDocuments">
  <documentRelationship type="inbound">document_contract3__cr</documentRelationship>
  <operator>all_equal</operator>
  <documentStateType>base:general:steady</documentStateType>
  <errorMessage>All related documents referencing this contract record must be in the steady state</errorMessage>
</action>
```

Three casing traps, all different from the function tree:

- `type` on `<documentRelationship>` is lowercase (`inbound`), where the function tree's direction
  is uppercase (`INBOUND`).
- `<operator>` is `all_equal`, lowercase and singular, where the method is `ALL_EQUALS`.
- `<documentStateType>` is colon delimited (`base:general:steady`), not dot delimited.

## After it runs

Retrieve the lifecycle again and find the criterion. A `SUCCESS` with the criterion missing or
unchanged is the silent no-op in SKILL.md Step 7. Then have the user try the state change on a
test record: the XML being accepted does not prove the check does what was asked.
