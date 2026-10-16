# Custom Pages: Styling, Runtime Behavior, and Naming

## Data crossing the Java-to-client boundary

Whatever `onLoad` passes to `.withData(...)` is a `JsonObject`, so the Vault field type to
`ValueType` mapping and the list of types `JsonObjectBuilder.setValue` accepts apply here exactly as
they do in any other Vault Java SDK code; both live in the java-sdk skill's
`references/platform-constraints.md`, under "Java SDK data types: VQL results and JSON output".

## Shadow DOM styling

Custom Pages mount inside a shadow root attached to a host element on the main Vault document. The
isolation is deliberate: it keeps page code and Vault's own styles from colliding. Two consequences
follow, both stated in Vault's Custom Pages docs. The runtime does not support `@font-face`. And
style written into `document.head`, whether by a `useEffect` that appends a `<style>` tag there, an
external `<link rel="stylesheet">`, or a UI library that targets `document.head` by default, stays
outside the shadow boundary and never reaches the page. When that happens the page renders as plain
HTML with no console message explaining it, so plan the styling approach up front rather than
diagnosing it after deploy.

**Attach CSS the supported way, in this order:**

1. List CSS files in `distribution-manifest.json`'s `stylesheets` array. This is the documented,
   supported way to attach a stylesheet to a Custom Page.
2. For CSS that has to be generated or conditional, render a `<style>` tag as a JSX child so the
   element lives inside the shadow tree, instead of writing to `document.head`:

```jsx
<div className="root">
  <style>{`.root { font-family: Roboto, sans-serif; }`}</style>
  <div className="content">...</div>
</div>
```

Roboto is Vault's font, so match it unless the page has a reason to use something else.

With esbuild, `import './page.css'` in the entry file writes a CSS file next to the bundle
(`dist/index.css` for an `index.js` output), and that file is what goes in `stylesheets`. A few
layout rules decide whether the page looks finished:

- Set `box-sizing: border-box` on everything. Under the default content-box, padding and borders
  add to every card's width, and a row of cards that fit in the design runs off the screen.
- A wide row centered with `justify-content: center` clips its left edge once it overflows.
  Give the row `width: max-content` inside a scrolling container instead.
- Size the layout to the viewport. Anything below the fold or past the right edge is missed in a
  screenshot and in a demo.

**Checking where the styles landed:** the DOM shows an `<iframe name="{page}__c">`, but it is empty
and the React tree is not inside it, so `iframe.contentDocument` answers nothing. Walk for the real
shadow root instead:

```js
const findShadow = (node) => {
  if (node.shadowRoot) console.log('shadow on', node.tagName, node.shadowRoot);
  for (const c of node.children) findShadow(c);
};
findShadow(document.documentElement);
```

If the rendered root is not inside that shadow tree, that is the cause. Confirm this before looking
at esbuild config, CSP, CSS specificity, or a blocked external resource.

## UI library integration inside the shadow root

A component library that injects its own styles or portals to `document.body` meets the same
boundary as above. Veeva documents per-library wiring at
`https://general.veevavault.dev/custom-pages/guides/ui-libraries/`; the shape in each case is
"redirect the library's mount/style target from `document`/`document.body` to the page's
`element`":

- **Emotion/MUI**: `createCache({ key, container: element })`, where container is `element`, not
  `element.getRootNode()` or `document.head`; wrap in `<CacheProvider>`. Drop `CssBaseline` (it
  targets `html`/`body`, which do not exist in the shadow root). Portal components (`Dialog`,
  `Modal`, `Popover`, `Menu`, `Select`/`Autocomplete` dropdowns, `Tooltip`, `Snackbar`, `Drawer`)
  default to `document.body`; pass `container={element}` to each or they render nothing visible.
- **styled-components**: `<StyleSheetManager target={stylesRoot}>`, where `stylesRoot` is a
  `<div>` appended to `element`.
- **Chakra or any library assuming a real `<html>`**: apply its root-level attributes to a wrapper
  `<div>` instead (`<div data-theme="light" style={{ colorScheme: 'light' }}>`), since the shadow
  tree has no `<html>` element.

Verify current specifics against the Custom Pages UI-libraries guide before committing to one:
library integration details shift with library major versions independent of the Vault release.

## Naming discipline

- The distribution name and every page name end `__c`, in `distribution-manifest.json` and in the
  MDL `Page` component alike.
- `url_path_name` must be URL-safe, unique across the Vault, and 40 characters or fewer.
- The MDL `Page` reaches the client code through the manifest: `client_distribution` names the
  distribution in `distribution-manifest.json`, `page_client_code` names a page inside that same
  manifest, and `page_controller` names the fully qualified Java class. All three have to match
  their targets character for character, or a freshly deployed page 404s or loads with no data.
- For MDL syntax, file naming, and component conventions beyond the `Page` attributes above, use
  the vault-administrator plugin's mdl skill.
