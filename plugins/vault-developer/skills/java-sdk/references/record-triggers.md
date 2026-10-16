# Record Triggers

A record trigger fires on its own, as the *System* user, whenever a data operation (UI or API)
inserts, updates, or deletes a record on the configured object. Nobody picks it from a menu and no
schedule kicks it off. This reference covers the whole shape: deciding a trigger is the right
entry point, the inputs needed before writing one, the annotation and event wiring, the canonical
template, and the patterns that keep the body bulk-safe.

## Is a record trigger the right entry point

Two other SDK entry points get asked for under the same "automate something on a record" framing.
A **record action** (`RecordAction`) is invoked explicitly, by a user or an API caller, against
specific records on demand, so "the user clicks a button" or "call this action via API" means
record action. A **job** (`Job`) covers scheduled or bulk background processing that is not tied
to one record's insert, update, or delete event, so "run this nightly" or "process every record
matching X in bulk" means job. The entry point table in the skill body has the rest.

If the request is genuinely one of those, say so plainly and write that entry point instead of
forcing the logic into a trigger shape.

Two more feasibility checks before writing:

- Trigger code runs as the *System* user, so it sees every record and field regardless of the
  invoking user's permissions. Logic that is supposed to depend on who did the save has to read
  the initiating user from the context, not rely on access failing.
- Up to 10 triggers can execute for a given object and event. If the object already carries
  several, the new logic may belong in an existing trigger rather than an eleventh class.

## Inputs to gather

Ask for whatever is missing in a single message, not one at a time.

| # | Field | Example | Safe default if skipped |
|---|-------|---------|--------------------------|
| 1 | **Object API name** | `product__v`, `study__v` | _none, must be provided_ |
| 2 | **Trigger events** | `BEFORE_INSERT`, `AFTER_UPDATE` | `BEFORE_INSERT, BEFORE_UPDATE` |
| 3 | **Fields involved** | `status__v`, `expected_date__c` | placeholder `TODO_FIELD__C` |
| 4 | **Business logic summary** | "Default expected_date to a week out on create" | `// TODO: implement business logic` |
| 5 | **Class name** | `ProductFieldDefaults` | Derived: `{ObjectPascalCase}{Purpose}` |

Object API names carry a namespace suffix (`__v` standard, `__c` custom, `__sys` system). If the
developer names the object loosely, ask instead of guessing the suffix. The object API name has no
safe default; nothing else can proceed without it.

## Package and class shape

A custom record trigger must live under the `com.veeva.vault.custom` package. Sub-packages are
fine for organization (`com.veeva.vault.custom.triggers`, or a feature-scoped path like
`com.veeva.vault.custom.rim.submissions.triggers`); only use one the developer actually asked
for or that already exists in the project.

The class declaration must include the `public` modifier and `implements RecordTrigger`. Name it
for the object and what it does: `ProductFieldDefaults`, `StudyArmCreator`, `PersonSkillAudit`,
not a generic `MyTrigger`. The class name becomes part of the deployed component's identity and
can't be renamed after deploy without a migration step.

## Canonical template

```java
package com.veeva.vault.custom.triggers;

import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.core.ValueType;
import com.veeva.vault.sdk.api.data.Record;
import com.veeva.vault.sdk.api.data.RecordChange;
import com.veeva.vault.sdk.api.data.RecordEvent;
import com.veeva.vault.sdk.api.data.RecordTrigger;
import com.veeva.vault.sdk.api.data.RecordTriggerContext;
import com.veeva.vault.sdk.api.data.RecordTriggerInfo;

@RecordTriggerInfo(object = "product__v", events = {RecordEvent.BEFORE_INSERT, RecordEvent.BEFORE_UPDATE})
public class ProductFieldDefaults implements RecordTrigger {

    public void execute(RecordTriggerContext context) {
        // Build any lookups/batched service calls ONCE, outside this loop, before iterating.
        for (RecordChange change : context.getRecordChanges()) {
            Record newRecord = change.getNew();
            String expectedDate = newRecord.getValue("expected_date__c", ValueType.STRING);
            if (expectedDate == null) {
                // business logic here
            }
        }
    }
}
```

## @RecordTriggerInfo attributes

| Attribute | Required | Notes |
|---|---|---|
| `object` | Yes | Valid Vault object API name, e.g. `product__v`. |
| `events` | Yes | One or more `RecordEvent` values in a `{}` array. |
| `order` | No | `TriggerOrder.NUMBER_1` through `NUMBER_10`. Lower runs first. Omit unless multiple triggers exist on the same object + event; unset defaults to running last. |

## RecordEvent values and data availability

`getNew()`/`getOld()` throw `RuntimeException` when called on an event where that data isn't
available; this is the most common runtime failure in a first-draft trigger. Guard with
`context.getRecordEvent()` whenever a trigger is wired to multiple events with different
availability.

| Event | getNew (get/set) | getOld (get/set) |
|---|---|---|
| `BEFORE_INSERT` | get + set | n/a |
| `AFTER_INSERT` | get only | n/a |
| `BEFORE_UPDATE` | get + set | get only |
| `AFTER_UPDATE` | get only | get only |
| `BEFORE_DELETE` | n/a | get only |
| `AFTER_DELETE` | n/a | get only |

`setValue()` only ever applies in `BEFORE_INSERT`/`BEFORE_UPDATE`: setting a value after the
record is already persisted (`AFTER_*`) or on a record being deleted has nothing to write to.
Trigger code runs as the *System* user: no record- or field-level security applies, so every
record and field is readable regardless of the invoking user's permissions.

`BEFORE` triggers can change field values, but those changes aren't in the database or VQL index
yet: `QueryService` inside the same `BEFORE` trigger only sees old values; use
`RecordService#readRecord` if you need a value another `BEFORE` trigger just set in the same
transaction. `AFTER` triggers run after the write and VQL indexing, so `QueryService` sees current
data there.

## Common patterns

### Field default (BEFORE event)
```java
String rating = newRecord.getValue("rating__v", ValueType.STRING);
if (rating == null || rating.isEmpty()) {
    newRecord.setValue("rating__v", "beginner__v");
}
```

### Validation: cancel the save (BEFORE event)
```java
String value = newRecord.getValue("required_field__v", ValueType.STRING);
if (value == null || value.isEmpty()) {
    change.setError("MISSING_REQUIRED_FIELD", "required_field__v must have a value.");
}
```
`RecordChange#setError` on a `BEFORE` event cancels the save for that one record; on `AFTER` it
throws `RuntimeException` instead. On a nested trigger it only cancels that nested record, not the
top-level operation; use `RollbackException` to cancel the entire transaction. `setError` is the
only exception-like mechanism trigger code can use besides throwing `RollbackException` directly;
no other exception type may be thrown or caught in Vault Java SDK code.

### Create related records in bulk (AFTER event)
Never query or save inside the per-record loop. Collect everything the loop needs first, make one
batched call, then distribute results:
```java
RecordService recordService = ServiceLocator.locate(RecordService.class);
List<Record> newRecords = VaultCollections.newList();

for (RecordChange change : context.getRecordChanges()) {
    Record study = change.getNew();
    Record arm = recordService.newRecord("study_arm__v");
    arm.setValue("study__v", study.getValue("id", ValueType.STRING));
    arm.setValue("status__v", "active__v");
    newRecords.add(arm);
}

RecordBatchSaveRequest saveRequest = recordService.newRecordBatchSaveRequestBuilder()
    .withRecords(newRecords)
    .build();

recordService.batchSaveRecords(saveRequest)
    .onErrors(errors -> errors.stream().findFirst().ifPresent(error -> {
        throw new RollbackException(
            "RELATED_RECORD_SAVE_FAILED",
            "Unable to create study_arm__v: " + error.getError().getMessage());
    }))
    .execute();
```

### DELETE event: use getOld(), never getNew()
```java
@RecordTriggerInfo(object = "study__v", events = {RecordEvent.BEFORE_DELETE})
public class StudyCleanupTrigger implements RecordTrigger {
    public void execute(RecordTriggerContext context) {
        for (RecordChange change : context.getRecordChanges()) {
            Record deletedRecord = change.getOld(); // getNew() throws on DELETE
            String studyId = deletedRecord.getValue("id", ValueType.STRING);
            // cleanup logic here
        }
    }
}
```

### Multi-event trigger with an event guard
```java
RecordEvent event = context.getRecordEvent();
if (event == RecordEvent.BEFORE_DELETE) {
    for (RecordChange change : context.getRecordChanges()) {
        Record oldRecord = change.getOld();
        // deletion-specific logic
    }
} else {
    for (RecordChange change : context.getRecordChanges()) {
        Record newRecord = change.getNew();
        // insert/update logic
    }
}
```

## Trigger order and nesting

Up to 10 triggers can execute for a given object + event; `order` breaks ties among them. A
trigger that performs a data operation (insert/update/delete) or role assignment can cause other
triggers to fire as nested triggers, up to 10 levels deep. All of `BEFORE`, `AFTER`, and every
nested level share the same request-wide time and memory budget; see the Limits table in
`references/platform-constraints.md`.

## Loads and migration mode

If the business logic implies this trigger should, or must not, run during a bulk load, say so in
the summary. The mechanics (Record Migration Mode does not suppress triggers; only
`X-VaultAPI-NoTriggers` does; `RecordTriggerContext#isMigrationModeEnabled()`) are in
`references/platform-constraints.md` § Record Migration Mode and No Triggers.

## Before handing the file back

- Confirm the package declared in the file matches the directory it was written to.
- For a trigger touching DELETE events, confirm the `getOld()`/`getNew()` guard is in place.
- Confirm the loop makes no per-record service call. If the logic could not avoid one, say so
  plainly rather than hiding it.
- Deployment is a separate job: the developer packages and deploys with their own tooling or the Vault UI.
