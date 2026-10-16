# Custom Pages: React, JavaScript, and esbuild Worked Example

One optional client stack, end to end. Everything in this file is a choice, not a requirement.
Vault's own Custom Pages quickstart uses React, plain JavaScript, and esbuild, which is why this
skill defaults to it, but TypeScript works, vanilla JavaScript works, any other bundler works, and
bundling itself is recommended rather than required. Swap the stack freely. The five pieces in
`scaffold-template.md` do not change.

Project layout (adapt to an existing repo's layout if one exists). In a Vault CLI workspace the
client source stays outside the workspace and builds into it, because the CLI zips everything under
`custom_pages/{distribution_name}/` as the distribution:

```text
/project-root
  client/{page-name}/
    src/{page-name}.jsx         # definePage() entry point (source)
    esbuild.mjs                 # bundling config
    package.json                # @veeva/vault, react, react-dom, esbuild
  workspace/                    # the Vault CLI workspace
    custom_pages/{distribution_name}/
      dist/{page-name}.js       # built entry point, produced by esbuild
      distribution-manifest.json
    javasdk/com/veeva/vault/custom/pages/
      {PageName}Controller.java
```

Outside a workspace, a Maven-style layout works as well: `dist/` and the manifest next to the
source, and the controller under `src/main/java/`.

## Setting up the toolchain

Node.js and npm installed globally, plus a sandbox Vault with Admin and code permissions. From the
project root:

```bash
mkdir -p client/{page-name} && cd client/{page-name}
npm init -y

# Vault Web SDK: install the tarball for the release being targeted. Current versions are
# listed at general.veevavault.dev/custom-pages/vault-web-sdk
npm install https://vault-web-sdk-releases-public.s3.us-west-2.amazonaws.com/main/veeva-vault-26.2.0-release.1.3.5.tgz

npm install react react-dom
npm install --save-exact --save-dev esbuild
```

## `src/{page-name}.jsx`

```jsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import { definePage } from '@veeva/vault';

export default definePage(({ data = {}, element, pageContext, sendEvent }) => {
    const { userId = '' } = data;
    const root = createRoot(element);
    root.render(<div>Hello user {userId}!</div>);
});
```

## `esbuild.mjs`

```js
import * as esbuild from 'esbuild';

await esbuild.build({
    entryPoints: ['src/{page-name}.jsx'],
    bundle: true,
    minify: true,
    sourcemap: true,
    outdir: '../../workspace/custom_pages/{distribution_name}/dist',
    format: 'esm',
});
```

`outdir` plus the entry point's file name is what `pages[].file` in the manifest has to point at:
`src/{page-name}.jsx` builds to `dist/{page-name}.js`, relative to the manifest. The distribution
holds minified output, with source maps included as separate map files, as the Vault docs
recommend for debugging. Watch the distribution size against the documented 50 MB limit; an
unminified bundle runs to several megabytes for one small page.

## Build and deploy shape (reference only; the developer runs these, this skill does not)

```bash
# client bundle, built into the workspace
cd client/{page-name}
node esbuild.mjs
```

Deployment goes through the workspace loop: the built folder, the controller under `javasdk/`, and
the `Page` MDL under `components/` go up together with `vault config push`, as the
vault-administrator plugin's config and mdl skills lay out. This skill runs none of it against a
Vault. To move the page between Vaults as a package, one VPK carries the built
`dist/` and `distribution-manifest.json` under `websdk/{distribution name}/`, the controller under
`javasdk/src/main/java/` by package path, and the `Page` MDL as a component step, imported and
deployed with Import Package and Deploy Package.

## Local dev loop

There is no official local-preview tool for Custom Pages client code; verify against current
Custom Pages docs before relying on one. A workable pattern: build a thin
local HTTP proxy that forwards `fetch` calls to the real Vault API using a stored session, and
branch the client's fetch calls on `window.location.hostname === 'localhost'` during development
so the same file works against the proxy locally and against `vaultApiClient` once deployed. This
shortens the edit-rebuild-redeploy loop for pure client changes but does not exercise the actual
`PageController`: any server-side change still needs a real deploy to test.
