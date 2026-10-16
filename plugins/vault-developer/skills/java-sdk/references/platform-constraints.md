# Platform Constraints for Vault Java SDK Code

These are
checked at **deploy-time validation**, not by local compilation: code that ignores them compiles
clean and still gets blocked on deploy (status *Blocked*), or fails at runtime if it slips past
validation. Validate early and often with the Validate Package endpoint rather than finding out on
a real deploy attempt.

A single deploy caps out at 1,000 `.java` source files or 50MB of data, whichever comes first; go
past either one and the whole deploy fails rather than partially applying
(`general.veevavault.dev/vault-sdk/deploying-code/create-vpk`). There's no separate per-class or
per-Vault limit beyond that source-file count.

## JDK allowlist

Only Vault Java SDK interfaces (`com.veeva.vault.sdk.api.*`) and a limited, explicitly allowlisted
set of JDK classes, interfaces, and methods may be referenced: `String`, `LocalDate`, `List`, and
similar common types are in; most of the rest of the JDK is not. Third-party libraries are never
allowed. The full allowlist ships with this skill as `references/jdk-allowlist.csv`
(`Class,Method` rows, allowed at the method level). For any JDK class or method you are unsure
about, grep that file before relying on it rather than assuming it's available. A class being
present does not mean every one of its methods is: `LocalDate.toString()` is not on the list,
so a date becomes a string with `date.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"))`. The
Vault Toolbox IntelliJ plugin's SDK
inspections catch most allowlist violations at edit time, before a deploy attempt.

The CSV is a copy of the published allowlist on the Vault Developer Portal
(Vault Java SDK > Limits & Restrictions > JDK Allowlist). If a deploy
validation rejects something the file says is allowed, re-check the portal copy.

## Forbidden constructs (validation-time restrictions)

| Construct | Why it's blocked | Use instead |
|---|---|---|
| `new ArrayList<>()`, `new HashMap<>()`, `new HashSet<>()`, and other direct JDK collection constructors | Vault Java SDK doesn't allow instantiating standard JDK collection classes | `VaultCollections.newList()` / `newSet()` / `newMap()` / `asList(...)` |
| `java.util.stream.Collectors` (`toList()`, `toSet()`, `toMap()`) | Same restriction, stream-collector form | `VaultCollectors.toList()` / `toSet()` / `toMap(...)` |
| `java.lang.reflect.*`, `Class.forName()`, `getClass().getMethod()`/`getDeclaredField()`/etc. | Runtime reflection isn't permitted in the sandbox | `Record.getValue(field, ValueType)` / `setValue(field, value)` for field access |
| `Thread`, `ThreadLocal`, `java.util.concurrent.*`, `synchronized`, `Thread.sleep()` | The SDK runtime is single-threaded; concurrency primitives cause unpredictable behavior or a deploy error | `ServiceLocator.locate(RuntimeService.class).sleep(ms)` for delays; `JobService` for background work |
| `java.io.File`/`FileInputStream`/etc., `java.nio.file.*` | No filesystem access in the sandbox | `RecordService` / `DocumentService` for persistence |
| `System.out` / `System.err` (including `System.out.println`) | Output goes nowhere in Vault | `LogService` for all logging and diagnostics (usage in `references/logging-and-debugging.md`) |
| `System.getenv()`, `System.getProperty()` | Silently return `null`/empty in the sandbox, not the host environment | Store configuration on a Vault object record instead |
| `System.exit()`, `Runtime.exec()`, `Runtime.halt()` | Process control isn't permitted | Remove the call; not needed in SDK code |
| Direct network sockets, external process execution | Bypasses Vault's security model | HTTP Callout / Spark Messaging for external integration |
| Non-`final` `static` fields, `volatile` fields, static fields built by a call | SDK code must be thread-safe; static state must be immutable, and static field initializers and init blocks aren't allowed | Keep static fields to `static final` compile-time constants (`private static final String NAME_FIELD = "name__v";`). Something like `static final DateTimeFormatter F = DateTimeFormatter.ofPattern(...)` fails validation ("Class field [F] must be an allowed type"); build it as a local inside the method |
| Subclassing an extension class; class constructors/init blocks on extension classes | Only extending Vault Java SDK interfaces is permitted | Implement the SDK interface directly (e.g. `implements RecordTrigger`) |
| Catching or throwing any exception type other than `RollbackException` | Only `RollbackException` is a recognized exception type in SDK code; it also can't be swallowed | Use `RollbackException` to roll back the transaction and set a user-facing message; use `RecordChange#setError` for a per-record `BEFORE`-event cancellation instead of an exception |
| Method names `notify`, `notifyAll`, `wait`, `readObject`, `writeObject`, `finalize`, `registerNatives` | Reserved by the platform; any reference fails on upload | Pick a different method name |
| `~` Method references (`ClassName::method`) | Rejected at deploy in field practice per internal SDK review guidance; the public restrictions list does not name them, so treat this as a practice rule and verify on your vault if it matters | Use an explicit lambda: `.map(each -> each.getUrl())`, not `.map(UrlResponse::getUrl)` |

## Bulk-safety limits (why the loop body matters)

| Category | Limit |
|---|---|
| Triggers per object + event | 10 |
| Nested trigger depth | 10 levels |
| Max request execution time | 100 seconds (all BEFORE/AFTER/nested triggers and action triggers in one transaction share this budget) |
| Max CPU execution time | 10 seconds |
| Max retry window | 6 hours |
| Max array size (e.g. doctype trigger version changes) | 500 elements |
| Max items in a `VaultCollections` List/Set/Map | 10,000 |
| Max source files per Vault | 1,000 |
| Max messages in a queue | 20,000 |
| Max memory, top-level extension | 40MB (nested extensions share/deallocate against this budget as they complete) |
| Max gross memory, whole transaction | 400MB |

Every entry point hands the class a list, not a single item: `getRecordChanges()` on a trigger,
`getRecords()` on a record action, `getDocumentVersions()` on a document action, the task items on
a job. Code that calls `QueryService`, `RecordService`, `DocumentService`, `HttpService`, or any
other Service inside a loop over that list multiplies service calls by item count and is the
single most common way to blow through the 100-second or 10-second CPU ceiling on a bulk API call,
an import, or a multi-select UI action: it passes testing against one item and fails in
production. The shape that survives is the same whatever the service:

1. Loop once to collect what the call needs: ids, a `CONTAINS` list, a JSON payload. Builders
   (`JsonService`, `VaultCollections`) are local and fine inside this loop; platform services are
   not.
2. Make one service call.
3. Loop again to apply the result back per item.

For `QueryService` that is one query with a `CONTAINS (${Custom.ids})` condition rather than one
query per id (SKILL.md covers `TokenService` for the list). For `RecordService` it is one
`batchSaveRecords` or `batchDeleteRecords` request built from the collected list, up to 500
records per call.

`HttpService` deserves its own note, because "batch it" is less obvious there and the failure
mode is worse. Callouts run outside the SDK transaction and do not roll back
(`general.veevavault.dev/vault-sdk/sdk-integrations/http-callout/http-service`). A loop that sends
one request per document and throws `RollbackException` when the fifth one fails has already
delivered four messages the remote system keeps while Vault discards the action. The documented
best practice is never to put a callout in a loop: build one payload carrying every item and send
one request. A document action that notifies an external system looks like this:

```java
List<DocumentVersion> versions = context.getDocumentVersions();
JsonArrayBuilder items = jsonService.newJsonArrayBuilder();
for (DocumentVersion version : versions) {
    items.add(jsonService.newJsonObjectBuilder()
            .setValue("document_id", version.getValue("id", ValueType.STRING))
            .setValue("document_name", version.getValue("name__v", ValueType.STRING))
            .build());
}

FormHttpRequest request = httpService.newHttpRequestBuilder()
        .withConnectionName(CONNECTION)
        .withMethod(HttpMethod.POST)
        .withPath(MESSAGE_PATH)
        .withBody(items.build())
        .build();

httpService.sendRequest(request, HttpResponseBodyValueType.STRING)
        .onError(error -> {
            throw new RollbackException("OPERATION_NOT_ALLOWED",
                    "Notification failed: " + error.getMessage());
        })
        .execute();
```

When the remote endpoint only accepts one item per request, or its response is slow, the send
does not belong in the user's transaction at all. Record what needs sending and let a job or a
Spark message processor make the callouts, which is also what the docs recommend so the user is
not left waiting on the response.

## Record Migration Mode and No Triggers

Record Migration Mode, set via `X-VaultAPI-MigrationMode` or Vault Loader's Record Migration Mode
option, bypasses entry criteria, entry actions, validation rules, and reference constraints. It
does **not**, by itself, stop record triggers from firing. Suppressing them requires the separate
`X-VaultAPI-NoTriggers` header (usable together with Record Migration Mode), which bypasses all
system, standard, and custom triggers. Inside the trigger, `RecordTriggerContext#isMigrationModeEnabled()`
reports whether the context that invoked it was in Record Migration Mode, which is useful for
branching logic that should behave differently during a migration load, but that check tells you
nothing about whether the trigger fired at all in the first place.

Because `X-VaultAPI-NoTriggers` suppresses a trigger entirely during a load, don't rely on a
trigger as the only thing enforcing a rule that must always hold. And since NoTriggers is typically
paired with Record Migration Mode, which also bypasses entry criteria and validation rules, those
can't be the fallback either. Enforce a must-hold invariant somewhere the load cannot skip: correct
data at the source, or a post-load verification pass, so the solution still holds when triggers are
off.

## Java SDK data types: VQL results and JSON output

These rules apply to any Vault Java SDK code that reads a VQL result or builds JSON, not only to
triggers. Record actions, jobs, and Custom Page controllers all hit them the same way.

### Vault field type to `ValueType`

Confirm the field's actual MDL type first (`GET /api/{ver}/metadata/vobjects/{object}`, or the docs
MCP `search_api_reference`) rather than guessing: a mismatch throws
`SDK_ERROR: does not support get/set value with java type [...]` at runtime, not at compile time.

| Vault Field Type | `ValueType` | Returned Java type |
|---|---|---|
| Text / LongText / RichText / Link / Email / Phone / Time | `ValueType.STRING` | `String` |
| Yes/No | `ValueType.BOOLEAN` | `Boolean` |
| Number / Currency / Percent | `ValueType.NUMBER` | `BigDecimal` |
| Date | `ValueType.DATE` | `LocalDate` |
| DateTime | `ValueType.DATETIME` | `ZonedDateTime` |
| Picklist (incl. `TOLABEL(picklist) AS x`) | `ValueType.PICKLIST_VALUES` | `List<String>` |
| Object / Parent / ID reference | `ValueType.STRING` | `String` |

~This table reflects the SDK's documented data-type-map convention; re-check against the
`search_api_reference`/`search_sdk_javadocs` MCP tools for a field whose type is unusual or
release-specific before trusting it on a query that has to work the first time.

### Types `JsonObjectBuilder.setValue` accepts

`JsonObjectBuilder.setValue(String name, Object value)` accepts an instance of a type
`JsonValueType` defines, not arbitrary boxed Java types. A `private static final int MAX = 500;`
compiles cleanly and then throws `Unsupported JSON value type [java.lang.Integer]` at runtime,
because raw `int`/`Integer` is not a `JsonValueType`.

| Accepted by `setValue` / `add` | Not accepted |
|---|---|
| `String`, `Boolean` | `int`/`Integer`/`long`/`Long`, `Float`/`Double` |
| `BigDecimal` (JSON NUMBER) | `java.util.Date` (use `LocalDate`/`ZonedDateTime`) |
| `LocalDate`, `ZonedDateTime` | `Map`, raw `Object` |
| Nested `JsonObject`, `JsonArray` | (none) |

Convert before writing: `setValue("max", BigDecimal.valueOf(MAX))`. `ZonedDateTime`/`LocalDate`
need explicit formatting before they are meaningful to a client
(`dt.format(DateTimeFormatter.ISO_OFFSET_DATE_TIME)`). The same rule covers
`JsonArrayBuilder.add`: the value passed must be a `JsonValueType`-backed instance, same list as
above.

### Capturing a query result out of an `onSuccess` lambda

The lambda passed to `.onSuccess(...)` has already run by the time `.execute()` returns, but a
captured local still has to be effectively final. Use a single-element array as a mutable
container, which is the idiomatic form in SDK code rather than `AtomicReference`:

```java
QueryService queryService = ServiceLocator.locate(QueryService.class);
String[] fullName = { "" };   // seed a safe fallback so the value is never null

QueryExecutionRequest request = queryService.newQueryExecutionRequestBuilder()
        .withQueryString(vql).build();
queryService.query(request)
    .onSuccess(response -> response.streamResults().forEach(row -> {
        fullName[0] = row.getValue("name__v", ValueType.STRING);
    }))
    .execute();   // fullName[0] is populated after this line
```
