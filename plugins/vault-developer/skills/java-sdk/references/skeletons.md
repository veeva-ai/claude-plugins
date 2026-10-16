# Compile-ready skeletons

Two shapes that come up often outside record triggers. Both are complete files with their imports,
so no docs lookup is needed to get a first build. Rename the package, class, object, and fields to
the developer's case. The platform rules in `platform-constraints.md` apply to both.

## A PageController that runs one query and returns JSON

One query, one pass over the rows, one `JsonObject` handed to the client as `data`. Fields are read
with the `ValueType` that matches the Vault field type; a mismatch fails at runtime, not in the
compiler.

```java
package com.veeva.vault.custom.pages;

import com.veeva.vault.sdk.api.core.RollbackException;
import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.core.ValueType;
import com.veeva.vault.sdk.api.executeas.ExecuteAs;
import com.veeva.vault.sdk.api.executeas.ExecuteAsUser;
import com.veeva.vault.sdk.api.json.JsonArrayBuilder;
import com.veeva.vault.sdk.api.json.JsonObject;
import com.veeva.vault.sdk.api.json.JsonService;
import com.veeva.vault.sdk.api.page.PageController;
import com.veeva.vault.sdk.api.page.PageControllerInfo;
import com.veeva.vault.sdk.api.page.PageLoadContext;
import com.veeva.vault.sdk.api.page.PageLoadResponse;
import com.veeva.vault.sdk.api.query.QueryExecutionRequest;
import com.veeva.vault.sdk.api.query.QueryService;

import java.math.BigDecimal;

/**
 * Sends one row per item__c record to the page as data.records.
 */
@PageControllerInfo
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
public class ItemListController implements PageController {

    private static final String VQL = "SELECT name__v, quantity__c FROM item__c";

    @Override
    public PageLoadResponse onLoad(PageLoadContext context) {
        JsonService jsonService = ServiceLocator.locate(JsonService.class);
        QueryService queryService = ServiceLocator.locate(QueryService.class);

        JsonArrayBuilder records = jsonService.newJsonArrayBuilder();

        QueryExecutionRequest request = queryService.newQueryExecutionRequestBuilder()
                .withQueryString(VQL)
                .build();

        queryService.query(request)
                .onSuccess(response -> response.streamResults().forEach(row -> {
                    String name = row.getValue("name__v", ValueType.STRING);
                    BigDecimal quantity = row.getValue("quantity__c", ValueType.NUMBER);

                    records.add(jsonService.newJsonObjectBuilder()
                            .setValue("name", name == null ? "" : name)
                            .setValue("quantity", quantity == null ? BigDecimal.ZERO : quantity)
                            .build());
                }))
                .onError(error -> {
                    throw new RollbackException("OPERATION_NOT_ALLOWED",
                            "Item query failed: " + error.getMessage());
                })
                .execute();

        JsonObject data = jsonService.newJsonObjectBuilder()
                .setValue("records", records.build())
                .build();

        return context.newLoadResponseBuilder()
                .withData(data)
                .build();
    }
}
```

## An AI tool handler with no input, returning JSON

For a tool that takes no parameters and answers over a whole set in one query. The paired `Aitool`
carries an empty input schema (see the ai-agents skill's `references/mdl-shapes.md`). A failed query
comes back as a JSON failure the model can read, not as a thrown exception.

```java
package com.veeva.vault.custom.ai;

import com.veeva.vault.sdk.api.ai.AiToolTypeExecuteContext;
import com.veeva.vault.sdk.api.ai.AiToolTypeExecutionResponse;
import com.veeva.vault.sdk.api.ai.AiToolTypeRuntimeHandler;
import com.veeva.vault.sdk.api.ai.AiToolTypeRuntimeHandlerInfo;
import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.core.ValueType;
import com.veeva.vault.sdk.api.executeas.ExecuteAs;
import com.veeva.vault.sdk.api.executeas.ExecuteAsUser;
import com.veeva.vault.sdk.api.json.JsonArrayBuilder;
import com.veeva.vault.sdk.api.json.JsonObjectBuilder;
import com.veeva.vault.sdk.api.json.JsonService;
import com.veeva.vault.sdk.api.query.QueryExecutionRequest;
import com.veeva.vault.sdk.api.query.QueryService;

import java.math.BigDecimal;

/**
 * Returns every item__c record with its quantity, in one query.
 */
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
@AiToolTypeRuntimeHandlerInfo(dynamicToolSpec = false)
public class ItemSummaryToolHandler implements AiToolTypeRuntimeHandler {

    private static final int MAX_ITEMS = 2000;
    private static final String VQL = "SELECT id, name__v, quantity__c FROM item__c";

    @Override
    public AiToolTypeExecutionResponse onExecute(AiToolTypeExecuteContext context) {
        JsonService jsonService = ServiceLocator.locate(JsonService.class);
        QueryService queryService = ServiceLocator.locate(QueryService.class);

        JsonArrayBuilder items = jsonService.newJsonArrayBuilder();
        int[] count = {0};
        String[] failure = {null};
        boolean[] truncated = {false};

        QueryExecutionRequest request = queryService.newQueryExecutionRequestBuilder()
                .withQueryString(VQL)
                .build();

        queryService.query(request)
                .onSuccess(response -> response.streamResults().forEach(row -> {
                    if (count[0] >= MAX_ITEMS) {
                        truncated[0] = true;
                        return;
                    }
                    count[0]++;
                    String name = row.getValue("name__v", ValueType.STRING);
                    BigDecimal quantity = row.getValue("quantity__c", ValueType.NUMBER);

                    items.add(jsonService.newJsonObjectBuilder()
                            .setValue("item_record_id", row.getValue("id", ValueType.STRING))
                            .setValue("name", name == null ? "" : name)
                            .setValue("quantity", quantity == null ? BigDecimal.ZERO : quantity)
                            .build());
                }))
                .onError(error -> failure[0] = error.getMessage())
                .execute();

        JsonObjectBuilder result = jsonService.newJsonObjectBuilder();
        if (failure[0] != null) {
            result.setValue("responseStatus", "FAILURE");
            result.setValue("failureReason", "Item query failed: " + failure[0]);
        } else {
            result.setValue("responseStatus", "SUCCESS");
            result.setValue("item_count", BigDecimal.valueOf(count[0]));
            result.setValue("truncated", Boolean.valueOf(truncated[0]));
            result.setValue("items", items.build());
        }

        return context.newSuccessBuilder(AiToolTypeExecutionResponse.JsonResultBuilder.class)
                .withJson(result.build())
                .build();
    }
}
```

- A handler with no input never calls `context.getInput()`. A tool with parameters reads them with
  `JsonValueType` from `com.veeva.vault.sdk.api.json`.
- Counters and failure flags captured by a lambda have to be effectively final, which is why they are
  one-element arrays.
- Both `onError` handlers report `error.getMessage()` and nothing more. Asking the error for its type
  and calling `name()` on that enum fails deploy validation, since `java.lang.Enum` has no entries on
  the JDK allowlist. That allowlist is the Java SDK "Limits and Restrictions" page in the Vault docs
  (the docs MCP, source `vault_developer_documentation`).
- Neither query takes a list. When one does, match it with `CONTAINS` and a `TokenService` token, as
  Step 5 of `SKILL.md` shows; `IN` takes only a subquery.
- The imports are the whole list. `@ExecuteAs` and `ExecuteAsUser` come from `...api.executeas`, and
  the AI types from `...api.ai`.
