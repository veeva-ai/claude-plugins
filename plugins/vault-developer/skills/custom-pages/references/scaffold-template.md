# Custom Pages: Scaffold Template

## What every Custom Page needs

Five pieces, in the order Vault resolves them:

1. **Client code** with a `definePage()` export, built to one JavaScript entry point per page.
2. **`distribution-manifest.json`**, naming the distribution and each page, and pointing at the
   built entry point for each one.
3. **The distribution deployed** to Vault: in a workspace, the built files sit in
   `custom_pages/{distribution_name}/` and go up with the workspace.
4. **A Java `PageController`** deployed to Vault, from `javasdk/` in a workspace.
5. **An MDL `Page` component** that links them.

The `Page` component reaches the client code through the manifest, not through a file path:
`client_distribution` names the distribution in `distribution-manifest.json`, `page_client_code`
names a page inside that same manifest, and `page_controller` names the fully qualified Java class.
Each of those names has to match its target exactly, or the page 404s or loads with no data.

The five pieces above are mandatory for every page. What is optional is the stack that produces
them: the skill defaults to React, JavaScript, and esbuild, but TypeScript, vanilla JavaScript, or
any other bundler works, and bundling itself is recommended
rather than required. The steps specific to that default choice (writing the client code as a JSX
component, the `npm install` of those packages, and the esbuild build) live in
`react-esbuild-example.md`. Swap the example freely; the five pieces do not change.

## 1. MDL `Page` component

```text
RECREATE Page {page_name}__c (
   label('{Page Label}'),
   active(true),
   client_distribution('Clientdistribution.{distribution_name}__c'),
   page_client_code('Pageclientcode.{page_name}__c'),
   page_controller('Pagecontroller.com.veeva.vault.custom.pages.{PageName}Controller'),
   url_path_name('{page-name}')
);
```

- `client_distribution` takes the `name` from `distribution-manifest.json`; `page_client_code` takes
  a `pages[].name` from that same manifest; `page_controller` takes the fully qualified class name
  of the Java controller.
- `client_distribution` and `page_controller` are required once the page name ends `__c`.
  `page_client_code` is required once `client_distribution` is set.
- `url_path_name` must be URL-safe and unique across the Vault, is max 40 characters, and becomes
  `https://{HOST}/ui/#custom/page/{url_path_name}`. Keep the component and distribution names
  URL-safe too: lowercase letters, digits, and underscores, ending `__c`.
- Do not add `default_client_code` or `server_code` to this component. Vault's own MDL reference
  marks both internal-Veeva-use-only on `Page`, and they belong to a different (Vault UI SDK) page
  mechanism, not Custom Pages.
- A `Tab` (nav bar entry), a `Subtab` under an existing tab, or a `Pagelink` (record-action
  navigation) can reference this `Page` afterward; draft one only after the developer picks it
  (SKILL.md Step 4 asks how users reach the page). A tab or subtab
  needs `url('https://${Vault.domain}/ui/#custom/page/${Page.url_path_name}')` next to
  `page('Page.{page_name}__c')`, or Vault refuses it with "Attribute url is required". The mdl
  skill's component-type map has the `Tab` / `Subtab` row.
- After a deploy, read back each of the three pieces at its own endpoint:
  `GET /api/mdl/components/Page.{page_name}__c` for the Page, `GET /api/{version}/code/{fully
  qualified controller class}` for the controller, and `GET /api/{version}/uicode/distributions`
  for the bundle. A controller does not show up in `vault_component__v`, so a query there
  proves nothing.

## 2. Java `PageController`

```java
package com.veeva.vault.custom.pages;

import com.veeva.vault.sdk.api.core.RequestContext;
import com.veeva.vault.sdk.api.core.ServiceLocator;
import com.veeva.vault.sdk.api.executeas.ExecuteAs;
import com.veeva.vault.sdk.api.executeas.ExecuteAsUser;
import com.veeva.vault.sdk.api.json.JsonObject;
import com.veeva.vault.sdk.api.json.JsonService;
import com.veeva.vault.sdk.api.page.PageController;
import com.veeva.vault.sdk.api.page.PageControllerInfo;
import com.veeva.vault.sdk.api.page.PageLoadContext;
import com.veeva.vault.sdk.api.page.PageLoadResponse;

@PageControllerInfo
@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)
public class {PageName}Controller implements PageController {

    @Override
    public PageLoadResponse onLoad(PageLoadContext context) {
        JsonService jsonService = ServiceLocator.locate(JsonService.class);

        JsonObject data = jsonService.newJsonObjectBuilder()
            .setValue("userId", RequestContext.get().getInitiatingUserId())
            .build();

        return context.newLoadResponseBuilder()
            .withData(data)
            .build();
    }
}
```

- `@PageControllerInfo` takes no attributes: it is a bare marker. The URL slug lives on the MDL
  `Page.url_path_name`, never on this annotation.
- `@ExecuteAs(ExecuteAsUser.REQUEST_OWNER)` runs the controller as the user who opened the page,
  respecting their permissions. Vault's Custom Pages docs specify this as the annotation a
  `PageController` must use; there is no documented case for running a page controller as `SYSTEM`.
- Whatever goes into `.withData(...)` becomes the client's `data` prop in `definePage()`.
- Add `onEvent(PageEventContext context)` (returns `PageEventResponse` via
  `context.newEventResponseBuilder()...`) only when the client calls `sendEvent(name, payload)`
  mid-session, e.g. a button that fetches more data without a full page reload.

### Loading Vault data in `onLoad`

The page-specific part is only the round trip: run the query, put the result into the load response,
read it as `data` on the client.

```java
// Add to the controller's imports above:
import com.veeva.vault.sdk.api.core.RollbackException;
import com.veeva.vault.sdk.api.core.ValueType;
import com.veeva.vault.sdk.api.query.QueryExecutionRequest;
import com.veeva.vault.sdk.api.query.QueryService;

// Inside onLoad(PageLoadContext context):
// Build `data` (a JsonObject) from your query first. The QueryService call, the effectively-final
// capture idiom, and JsonObject building all live in the java-sdk skill.
return context.newLoadResponseBuilder().withData(data).build();
```

`ValueType` and `RollbackException` come from `core`, and the two query types from `query`. None
of them is in `data`.

For VQL itself (syntax, quoting, `TOLABEL`, relationship traversal, operators), load the
vault plugin's query skill for every `SELECT`, simple or not, and run it once with `vault query`
before hand-off. For the `QueryService` call shape, the effectively-final
capture idiom, and the field-type to `ValueType` mapping that `getValue` needs, see the java-sdk
skill's `references/platform-constraints.md`. The query's `onError` handler throws
`RollbackException`; it is the only exception type SDK code may throw.

## 3. `distribution-manifest.json`

```json
{
    "name": "{distribution_name}__c",
    "pages": [
        { "name": "{page_name}__c", "file": "dist/{page-name}.js" }
    ],
    "stylesheets": ["styles/reset.css"]
}
```

- `name` and every `pages[].name` end `__c`. The MDL `Page` points at these two values through
  `client_distribution` and `page_client_code`.
- `pages[].file` is the path, inside the ZIP, to the **built** JavaScript entry point, not to the
  source file. A path like `src/{page-name}.jsx` here will not load.
- `pages[].export` is only needed for a named export; a default export omits it. When it is set it
  must match the export name in the built file exactly.
- `stylesheets` is optional; entries there get attached to the shadow-root document, which is the
  supported way to load CSS (see `gotchas-and-type-mapping.md` for why an in-code `<link>` does not
  work the same way).

## 4. Client entry point (`definePage`)

The one required shape: a module whose export is the value of `definePage()` from `@veeva/vault`,
or a function returning a promise that resolves to it. What renders inside the callback is the
developer's choice.

`@veeva/vault` is the Vault Web SDK. It installs from a Vault release tarball, not the public npm
registry, so install it and your framework packages before these imports resolve. The exact `npm
install` commands, including the tarball URL, are in `react-esbuild-example.md`.

```jsx
import { definePage } from '@veeva/vault';

export default definePage(({ data = {}, element, pageContext, sendEvent }) => {
    // mount the UI onto element
});
```

`definePage`'s callback receives:

| Param | What it is |
|---|---|
| `data` | whatever `onLoad`'s `.withData(...)` returned, JSON-decoded |
| `element` | the shadow-root host element; append/mount here, never to global `document` |
| `pageContext` | `{ reload }`, calls `onLoad` again |
| `sendEvent` | `(eventName, payload) => Promise<{ data? }>`, calls `onEvent`, up to 50 queued |

### Calling Vault API directly from the client

The normal path for Vault data is the controller round trip above: `onLoad` runs the query and the
client reads the result as `data`. A direct client-side API call is the exception, for the case
where a page has to reach Vault without going back through the controller.

```js
import { vaultApiClient } from '@veeva/vault';

const response = await vaultApiClient.fetch('/v26.1/query?q=SELECT+id+FROM+user__sys', {
    headers: { 'X-VaultAPI-ClientID': '{registered-client-id}' },
});
```

`vaultApiClient.fetch` targets the authenticated Vault automatically and counts toward normal API
rate limits. Users do not need direct API permission to view a page that calls Vault API this way.

Set an `X-VaultAPI-ClientID` on every call. Past traceability in the API Usage Logs, an Admin can
enable Client ID Filtering (Admin > Settings > General Settings); once it is on, Vault rejects API
requests whose client ID does not match an active Connection in that Vault, so a page's calls need
a client ID registered there. For endpoints, request and response shapes, auth, and rate limits, check the
Vault API reference through the docs MCP; the Admin setting itself is under Admin > Settings >
General Settings.

## Worked example

For one complete stack end to end (project layout, the npm install steps including the Vault Web
SDK tarball, the `esbuild.mjs` build config, the built entry point the manifest has to point at,
the build and deploy command shapes, and the local dev loop), see `react-esbuild-example.md`. It
uses React, plain JavaScript, and esbuild. That is the skill's default, not a requirement: any
framework, language, or bundler satisfies the five pieces above.
