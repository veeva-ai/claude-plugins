# Handler scaffolds

Java shapes for the handler types this skill generates. Adapt names and body logic to the user's
case rather than inventing a different shape. `~` marks anything the docs do not state.

## Before writing any of these

The Vault Java SDK platform rules apply to every class below and are not repeated here: the JDK
allowlist, `VaultCollections` in place of JDK collection constructors, `LogService` message-template
logging, `RollbackException` as the only throwable, the ban on broad `catch` and on regex,
`String.split`, and `String.format`, and the bulk-safety rule against a service call inside a
per-record loop. They belong to the java-sdk skill; read its `references/platform-constraints.md`
before writing a body.

Four things are specific to AI Agent handlers and worth carrying here:

- **`@ExecuteAs` and `ExecuteAsUser` import from `com.veeva.vault.sdk.api.executeas`**, not
  `...core`. The AI and agent types import from `com.veeva.vault.sdk.api.ai` and
  `com.veeva.vault.sdk.api.agent`. A wrong package compiles under plain `javac` and fails Vault's
  import with `cannot find symbol`.
- **Every handler carries two annotations**, `@ExecuteAs(...)` plus the handler-specific
  `@*RuntimeHandlerInfo` or `@AgentActionResultHandlerInfo`. Missing the second one is the usual
  cause of an import failing with "Implementing interface [...] is not allowed". Add the
  annotation rather than assuming an entitlement gap `~`.
- **The scope enums have to agree.** `AiScope` in the annotation, `AiScopeType` passed to
  `getScopeSource`, and the scope-source variable's type all name the same scope. A mismatch is a
  silent runtime bug, not a compile error.
- **Result-builder pairing is strict.** `JsonResultBuilder.class` goes with `.withJson(JsonObject)`
  and `TextResultBuilder.class` with `.withText(String)`. Never mixed.

For an interface's exact method set, inspect the SDK `.jar` the developer's project declares (a
Maven dependency or a `lib` folder), and fall back to the docs MCP
(`mcp__vault-docs-mcp__search_documentation`, source `vault_java_sdk_javadocs`, query the interface
name). Never search the disk or the home directory for a `.jar`.

## 1. AI Context Type handler

`@AiContextTypeRuntimeHandlerInfo(scope = AiScope.<DOCUMENT|OBJECT|ALL>)` pairs with
`context.getScopeSource(AiScopeType.<DOCUMENT|OBJECT>)`. `getScopeSource(...)` is optional when the
context only surfaces Vault-level metadata and never reads the scoped record or document; do not
call it just to match the shape. `AiScope.ALL` resolves its scope source at request time rather
than at authoring time. Scope is declared only on the Java annotation: `Aicontexttype` has no
`scope(...)` attribute.

```java
package com.veeva.vault.custom.context;

import com.veeva.vault.sdk.api.ai.*;
import com.veeva.vault.sdk.api.core.ValueType;
import com.veeva.vault.sdk.api.document.DocumentVersion;
import com.veeva.vault.sdk.api.executeas.ExecuteAs;
import com.veeva.vault.sdk.api.executeas.ExecuteAsUser;

/**
 * <One-line purpose, from the user's description.>
 */
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
@AiContextTypeRuntimeHandlerInfo(scope = AiScope.DOCUMENT)
public class MyContextHandler implements AiContextTypeRuntimeHandler {
    @Override
    public AiContextTypeResolveResponse onResolve(AiContextTypeResolveContext context) {
        AiDocumentScopeSource src = context.getScopeSource(AiScopeType.DOCUMENT);
        DocumentVersion doc = src.getDocumentVersion();
        String name = doc.getValue("name__v", ValueType.STRING);

        AiContextTypeOutputItem out = context.newOutputItemBuilder()
                .withText("The current document name is [" + name + "]")
                .build();

        return context.newSuccessResponseBuilder()
                .appendOutputItem(out)
                .build();
    }
}
```

A context that assembles a `JsonObject` still returns it through `withText(...)`: serialize with
`payload.asString()` first, because `withText` takes a `String`.

**Returning an error instead of a result.** `context.newErrorResponseBuilder()` takes no class
argument and pairs with `.withErrorType(...)` plus `.withErrorMessage(...)`. Pick the type
deliberately from `AiContextTypeResolveErrorType` so callers can tell a bad-input error from a
general execution failure.

**The SDK has no document-text-extraction API.** `DocumentSourceFileReference` exposes the binary
source file only; there is no `getText(...)` on a document version. The one path to extracted text
is a Vault API callout to
`GET /api/{version}/objects/documents/{doc_id}/versions/{major}/{minor}/text` through
`HttpService`, which costs real per-call latency next to an SDK-native read. Confirm with the user
that the callout is acceptable and that a local Connection exists before generating a handler that reads
document content this way. One binding choice is worth calling out: `withConnectionName("<api>__c")`
needs a matching local Connection record on the target Vault before `runtime_code` resolves at
import and runs as that record's authorized user, while `.withLocalConnection(...)` runs as the
current request's user and needs no Connection record. Pick on the security binding, not on
convenience. The rest of `HttpService` is the java-sdk skill's material.

**Parsing a version id.** Vault stores a document version reference as `<docId>_<major>_<minor>`,
for example `103_0_3`. Regex, `String.split`, and `String.format` are all blocked, so parse with
`indexOf` plus `substring`:

```java
private static class VersionIdParts { String docId, major, minor; }

private static VersionIdParts parseVersionId(String versionId) {
    if (versionId == null || versionId.isEmpty()) { return null; }
    int firstUnderscore = versionId.indexOf('_');
    if (firstUnderscore <= 0) { return null; }
    int secondUnderscore = versionId.indexOf('_', firstUnderscore + 1);
    if (secondUnderscore <= firstUnderscore + 1
        || secondUnderscore == versionId.length() - 1) { return null; }
    VersionIdParts p = new VersionIdParts();
    p.docId = versionId.substring(0, firstUnderscore);
    p.major = versionId.substring(firstUnderscore + 1, secondUnderscore);
    p.minor = versionId.substring(secondUnderscore + 1);
    return p;
}
```

MDL registration: see the `Aicontexttype` section of `mdl-shapes.md`.

## 2. AI Tool Type handler, static input spec

`@AiToolTypeRuntimeHandlerInfo(dynamicToolSpec = false)`. The input schema lives entirely in the
paired `Aitool`'s `input(...)` clause; the handler reads whatever that schema promised.

```java
package com.veeva.vault.custom.tooltype;

import com.veeva.vault.sdk.api.ai.*;
import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.executeas.ExecuteAs;
import com.veeva.vault.sdk.api.executeas.ExecuteAsUser;
import com.veeva.vault.sdk.api.json.*;

/**
 * <One-line purpose.>
 */
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
@AiToolTypeRuntimeHandlerInfo(dynamicToolSpec = false)
public class MyToolHandler implements AiToolTypeRuntimeHandler {
    @Override
    public AiToolTypeExecutionResponse onExecute(AiToolTypeExecuteContext context) {
        JsonObject input = context.getInput();
        String someArg = input.getValue("some_arg", JsonValueType.STRING);

        JsonService jsonService = ServiceLocator.locate(JsonService.class);
        JsonObjectBuilder result = jsonService.newJsonObjectBuilder();
        result.setValue("responseStatus", "SUCCESS");
        // ... do the work, query or save records as needed, build the rest of the result ...

        return context.newSuccessBuilder(AiToolTypeExecutionResponse.JsonResultBuilder.class)
                .withJson(result.build())
                .build();
    }
}
```

- **A tool with no input and a whole-set answer** has a compile-ready file, imports included, in the
  java-sdk skill's `references/skeletons.md`. Start from that one, with the empty `input(...)` schema
  from `mdl-shapes.md`, rather than from the sketch above.
- **A structured `responseStatus` / `failureReason` envelope is the better failure shape** for a
  JSON-result tool. Return a `SUCCESS` or `FAILURE` result rather than throwing, so the calling
  action's `instructions(...)` can branch on it in plain language. Reserve `RollbackException` for
  what the tool genuinely cannot recover from.
- **A tool that queries and disambiguates**, resolving a name to a unique record and failing
  distinctly on zero matches, several matches, or an already-existing link, is the common shape for
  a "do a thing on behalf of the user" tool. Escape user-supplied values with
  `queryService.escapeValue(...)` and stream results through `.onSuccess(...)`.

MDL registration: see the `Aitooltype` plus `Aitool` section of `mdl-shapes.md`. Both files are
required.

## 3. AI Tool Type handler, dynamic input spec

`@AiToolTypeRuntimeHandlerInfo(dynamicToolSpec = true)` requires implementing `onGenerateInput`.
The paired `Aitool` omits the `input(...)` clause; the schema is built here instead.

**`dynamicToolSpec` defaults to `true`.** A handler annotated bare lands in dynamic mode and has to
implement `onGenerateInput`, or Vault throws at runtime when the tool is called. Always write the
element explicitly, in both modes, rather than relying on the default.

```java
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
@AiToolTypeRuntimeHandlerInfo(dynamicToolSpec = true)
public class MyDynamicToolHandler implements AiToolTypeRuntimeHandler {
    @Override
    public AiToolTypeExecutionResponse onExecute(AiToolTypeExecuteContext context) {
        // same execute shape as the static example
    }

    @Override
    public AiToolTypeGenerateInputResponse onGenerateInput(AiToolTypeGenerateInputContext context) {
        LlmToolInputItem arg = context.newSimpleTypeLlmToolItemBuilder()
                .withType(LlmToolInputItemType.STRING)
                .withDescription("<what this argument means and its exact expected shape, "
                        + "for example a document id of form id_major_minor such as 100_1_0>")
                .build();

        LlmToolInputObject.Builder input = context.newObjectLlmToolItemBuilder();
        input.addProperty("some_arg", arg);
        input.addRequired("some_arg");

        return context.newSuccessBuilder()
                .withInput(input.build())
                .build();
    }
}
```

Dynamic mode is right when the input shape depends on live metadata, for example every `STRING`
field on an object. Build it with `ObjectMetadataService` inside `onGenerateInput` rather than
hardcoding property names. Every input item still needs a non-empty description, the same rule as
the static schema's JSON, just expressed through the builder.

Do not force a structured type through the simple builder. `SimpleTypeBuilder.withType(...)` fails
on `LlmToolInputItemType.ARRAY` and `LlmToolInputItemType.OBJECT`; the array and object builders
are in the SDK javadocs.

## 4. AgentActionResultHandler

Runs in a new SDK transaction after the agent action completes, so there is no closure state from
wherever `runAgentAction(...)` was called. The only way back to "what record or document was this
for" is `result.getScopeSource(...)`. The class has to live in its own file.

```java
package com.veeva.vault.custom.agent;

import com.veeva.vault.sdk.api.agent.*;
import com.veeva.vault.sdk.api.ai.AiObjectScopeSource;
import com.veeva.vault.sdk.api.ai.AiScopeType;
import com.veeva.vault.sdk.api.core.LogService;
import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.core.VaultCollections;
import com.veeva.vault.sdk.api.data.Record;
import com.veeva.vault.sdk.api.data.RecordService;

/**
 * Result handler for <agent name>'s <action name> action. Writes the returned text onto
 * <target_field__c> on the originating <object__c> record, reached via the OBJECT scope source.
 */
@AgentActionResultHandlerInfo
public class MyResultHandler implements AgentActionResultHandler {

    private static final int TARGET_FIELD_MAX_LENGTH = 5000; // match the field's actual max_length

    @Override
    public void onSuccess(AgentActionSuccess result) {
        if (result.getOutputSize() == 0) {
            return;
        }
        if (!AgentActionOutputType.TEXT.equals(result.getOutputType(0))) {
            return;
        }

        String output = result.getOutput(0, AgentActionOutputType.TEXT);
        if (output.length() > TARGET_FIELD_MAX_LENGTH) {
            output = output.substring(0, TARGET_FIELD_MAX_LENGTH);
        }

        AiObjectScopeSource scopeSource = result.getScopeSource(AiScopeType.OBJECT);
        Record record = scopeSource.getRecord();
        record.setValue("target_field__c", output);

        RecordService recordService = ServiceLocator.locate(RecordService.class);
        recordService.batchSaveRecords(recordService.newRecordBatchSaveRequestBuilder()
                        .withRecords(VaultCollections.asList(record))
                        .build())
                .rollbackOnErrors()
                .execute();
    }

    @Override
    public void onError(AgentActionError error) {
        LogService logService = ServiceLocator.locate(LogService.class);
        logService.error("Agent action failed: {} - {}", error.getType(), error.getMessage());
    }
}
```

For **document scope**, the pattern changes on two points:

- Reach the record indirectly: `result.getScopeSource(AiScopeType.DOCUMENT).getDocumentVersion()`,
  then read `version_id` as `ValueType.STRING`. Then query
  `SELECT id FROM <obj>__c WHERE <docref_field>__c = '<version_id>'` to find the originating
  records and save each one.
- `onError` never throws and never does write-back work. It logs and returns.

**Why `version_id` and not the three numeric version fields.** The instance handed to a result
handler is an immutable, sparsely populated document version, exposing `id`, the two numeric
version fields, and `version_id`. That `version_id` value is exactly what a
document-version-reference field stores on a record, so it is one read instead of three and needs
no reassembly `~`. If the handler produces no log output past a particular line and `getValue(...)`
was the last call before it, the SDK swallowed the failure, typically an unsupported field name on
the immutable variant: the debug log then shows the start and end markers milliseconds apart with
nothing between them `~`.

**The handler's log entries land in a different file from the action's** `~`. The result handler
runs on its own consumer thread pool, so its execution id and transaction id both differ from the
action's. When creating a debug log session for an SDK-driven agent flow, filter on both the action
or trigger class and the handler class, or you capture one side of the exchange only.

`onSuccess` must reference nothing captured from outside the handler, no instance field and no
value closed over from the code that called `runAgentAction`. Check for this before calling the
file done.

## 5. Record action or record trigger that starts an agent and writes back

Both build the same three-part flow: build a scope source for the record, start an agent instance,
run an action against it with the handler from section 4 wired in, then stop the instance. They
differ in how they are triggered and how they iterate records. The entry point itself, its
annotation, and its context object are the java-sdk skill's material; what follows is the agent
wiring inside it.

### Record action (user clicks a button)

```java
package com.veeva.vault.custom.actions;

import com.veeva.vault.custom.agent.MyResultHandler;
import com.veeva.vault.sdk.api.action.*;
import com.veeva.vault.sdk.api.agent.*;
import com.veeva.vault.sdk.api.ai.AiScopeSource;
import com.veeva.vault.sdk.api.ai.AiService;
import com.veeva.vault.sdk.api.core.*;
import com.veeva.vault.sdk.api.data.Record;

import java.util.List;

@RecordActionInfo(label = "<Action Label>", object = "<object__c>", usages = {Usage.USER_ACTION})
public class MyAgentAction implements RecordAction {

    private static final String AGENT_NAME = "<agent__c>";
    private static final String ACTION_NAME = "<agent_action__c>";
    private static final String PROMPT = "<user-input text sent to the agent>";

    @Override
    public void execute(RecordActionContext context) {
        AiService aiService = ServiceLocator.locate(AiService.class);
        AgentService agentService = ServiceLocator.locate(AgentService.class);
        LogService logService = ServiceLocator.locate(LogService.class);

        List<Record> records = context.getRecords();
        List<StartAgentInstanceRequest> startRequests = VaultCollections.newList();

        for (int i = 0; i < records.size(); i++) {
            String recordId = records.get(i).getValue("id", ValueType.STRING);

            AiScopeSource scopeSource = aiService.newAiObjectScopeSourceBuilder()
                    .withObjectName("<object__c>")
                    .withRecordId(recordId)
                    .build();

            startRequests.add(agentService.newStartAgentInstanceRequestBuilder()
                    .withAgentConfigurationName(AGENT_NAME)
                    .withScopeSource(scopeSource)
                    .build());
        }

        agentService.batchStartAgentInstances(startRequests)
                .onSuccesses(successes -> {
                    for (int i = 0; i < successes.size(); i++) {
                        String agentInstanceId = successes.get(i).getAgentInstanceId();

                        AgentActionInputItem inputItem = agentService.newAgentActionInputItemBuilder()
                                .addText(PROMPT)
                                .build();
                        AgentActionInput userInput = agentService.newAgentActionUserInputBuilder()
                                .appendInputItem(inputItem)
                                .build();
                        AgentActionParameters parameters = agentService.newAgentActionParametersBuilder()
                                .withAgentInstanceId(agentInstanceId)
                                .withActionName(ACTION_NAME)
                                .appendInput(userInput)
                                .withResultHandler(MyResultHandler.class)
                                .build();

                        agentService.runAgentAction(parameters);

                        // Stop once the last action is started: onSuccess and onError run in an
                        // independent later transaction either way.
                        StopAgentInstanceRequest stopRequest = agentService.newStopAgentInstanceRequestBuilder()
                                .withAgentInstanceId(agentInstanceId)
                                .build();
                        agentService.batchStopAgentInstances(VaultCollections.asList(stopRequest))
                                .onErrors(errors -> logService.error("Failed to stop agent instance {}: {}",
                                        agentInstanceId, errors.get(0).getError().getMessage()))
                                .execute();
                    }
                })
                .onErrors(errors -> logService.error("Failed to start agent instance: {}",
                        errors.get(0).getError().getMessage()))
                .rollbackOnErrors()
                .execute();
    }

    @Override
    public boolean isExecutable(RecordActionContext context) {
        return true;
    }
}
```

Instance reuse and shutdown timing follow the Best Practices on the docs' How to Execute Agent
Actions page; read it before changing this flow.

For **document scope**, swap the object-scope builder for
`aiService.newAiDocumentScopeSourceBuilder().withDocumentVersionId(<value from a
document-version-reference field, read as ValueType.STRING>).build()`.

### Record trigger (fires automatically on insert or update)

Same three-part flow inside an `@RecordTriggerInfo(object = "<object__c>", events =
{RecordEvent.AFTER_INSERT, RecordEvent.AFTER_UPDATE})` class, with four differences from the record
action above:

1. **Iterate `context.getRecordChanges()`**, not `context.getRecords()`. Read the scope key from
   `change.getNew()`.
2. **Never subscribe to a `BEFORE_*` event on a trigger that calls the agent SDK.** On
   `BEFORE_INSERT` the record has no id yet, so the async result handler cannot find it later, and
   the handler reaches a record save, which is not allowed inside any `BEFORE_*` trigger regardless.
3. **`change.getOld()` is event-gated.** Capture the firing event once at the top and only call
   `getOld(...)` inside an update or delete branch; the unavailable accessor throws at runtime and
   there is no return-null path. The availability table is in the java-sdk skill's
   `references/record-triggers.md`.
4. **A loop guard is required only when the result handler writes back to the originating record or
   another record on the same object.** Ask the user which write-back target applies. When it is
   required: on `AFTER_UPDATE`, compare the old and new scope-key values and skip starting an
   instance when they match, so the handler's own write-back does not re-fire the trigger. When the
   target is a different object, omit the guard. No loop is possible, and a stray guard silently
   swallows legitimate re-fires.

Also skip the `batchStartAgentInstances(...)` call when the accumulated request list is empty, and
never write to records inside the trigger's own `execute(...)`. That is the result handler's job,
in its own transaction.

MDL registration is what makes any of this reachable from the UI. A deployed record action becomes a
clickable user action through an `Objectaction` whose `action` attribute names the Java class as
`Recordaction.<package>.<ClassName>`, and a trigger needs its own component registration. Authoring
and executing that MDL is the vault-administrator plugin's mdl skill's job, so name it in the
handoff: the record action is not user-visible until that component exists.
