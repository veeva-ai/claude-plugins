---
name: vpk
version: "0.1.0"
description: >
  For a Vault admin or developer who needs to move configuration or code between Vaults as a
  package: build a VPK from a workspace or a set of components, import and deploy a VPK to a
  Vault, validate a package before it deploys, and read the deployment result. Use it whenever
  a feature has to go sandbox to sandbox or sandbox to production with its dependencies
  tracked, and whenever Java SDK code or a Custom Page bundle has to travel inside a package, since
  the CLI does not put Java in one. Also use it when a deploy failed partway and the user wants to know
  what landed.
---

# Vault Packages (VPK)

A VPK is a zip: a `vaultpackage.xml` manifest plus the component files and code it carries.
Vault imports it as a `vault_package__v` record, validates it against what the target already
has, and deploys it step by step. This skill builds packages, moves them, and reads the result.
The MDL inside a package is the mdl skill's job, the Java is java-sdk's, and the Custom Page
code is custom-pages'. Which components a feature needs comes from the config skill's
dependency walk, not from here.

`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` has the CLI commands and the rules every skill
follows when using it. Read it before the first CLI command in a task.

## Scope

Building, validating, importing, and deploying packages, and reporting what a deploy did.
Out of bounds:

- Authoring MDL: mdl.
- Writing or reviewing Java SDK source and Custom Page code: java-sdk and custom-pages.
- Deciding which components belong to a feature: config's "scoping a feature" walk.
- Creating or refreshing sandboxes: the Vault Admin UI.

## Handoffs

Sign-in and the production check come from the vault plugin's login skill. A deploy is a
write, so the check runs first, every time. When the package needs a component that does not
exist yet, hand the statement to mdl; when it needs code that does not compile, hand the
class to java-sdk. Name the skill and stop.

## Step 1: Check for the CLI

```bash
vault version
```

An answer of 26.2.4 or later means Steps 2 to 4 use the CLI. No answer, or older than 26.2.4, means
offer to install it first, per rule 3 in `reference/vault-cli.md`, and run the install only on
the user's yes. If they say no or it fails, Step 5 for the same work through the API, and do
not raise it again.

## Step 2: Build the package

From a workspace the config skill pulled:

```bash
MSYS_NO_PATHCONV=1 vault --verbose api "/objects/users/me" 2>&1 | grep -m1 '^> '   # the Vault it packages against
vault config package create --dir <workspace> --output <path>.vpk
```

It packages the workspace's differences from the CLI's current Vault, the host the first line
shows (the login skill's Step 2 has how to change it). Against the source Vault that is the change not pushed yet, so
build the package before pushing it there. Against the target it is everything the target holds
differently, which can be more than the change at hand. Say which Vault you built against, read
the component count in the output, and never hand over a package that reports zero.
`${CLAUDE_PLUGIN_ROOT}/reference/cli-interim.md`, when it is present, covers a package that comes
out empty.

The CLI packager fits a change that is still in a workspace. A whole feature that already works
in the source Vault comes out cleaner as that Vault's own export: Admin > Deployment > Outbound
Packages, Create with the type Migration, add the components, then View/Add Dependencies and
Export from the Actions menu. Vault posts a notification and sends an email with the download
link when the export finishes. An exported package recreates each component as the source has
it. When its Code section includes Vault Java SDK code, the deployment option defaults to Replace
All, which removes any code on the target that the package does not carry, so read
`<deployment_option>` in its manifest before it goes anywhere. Step 5 has the same export
through the API.

The CLI leaves `javasdk/` out of the package it builds. When the feature includes Java, say so
before building anything, and take one of two routes: export the package from the source
Vault as above, with Vault Java SDK code included in its Code section, or build the zip by
hand with the code under `javasdk/` next to the manifest. The manifest format is in the Vault
docs MCP (`mcp__vault-docs-mcp__search_documentation`, search "vaultpackage.xml"). A hand-built
package that carries a Custom Page looks like this:

```
vaultpackage.xml
components/00010/Page.org_chart__c.mdl
components/00010/Page.org_chart__c.md5        {md5 of the .mdl} Page.org_chart__c
javasdk/src/main/java/com/veeva/vault/custom/pages/OrgChartController.java
websdk/org_chart__c/distribution-manifest.json
websdk/org_chart__c/dist/index.js
websdk/org_chart__c/dist/index.css
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<vaultpackage xmlns="https://veevavault.com/">
  <name>org-chart-page</name>
  <source>
    <vault>{source vault id}</vault>
    <author>{username}</author>
  </source>
  <summary>Org Chart custom page</summary>
  <javasdk>
    <deployment_option>incremental</deployment_option>
  </javasdk>
</vaultpackage>
```

The client distribution goes in a top-level `websdk/{distribution name}/` folder with its
`distribution-manifest.json` at the root of that folder and the built files it names. The
manifest has no `websdk` element; `<javasdk><deployment_option>` covers both code folders. Vault
ignores a top-level folder it does not recognise, without an error, so a misnamed `websdk`
folder deploys a package with no client code in it. The deploy results should list a Web SDK
step next to the Java and Page steps.

Before handing the package over, open it and check it. Read everything from the package, not
from what the user asked for.

1. List what it carries: component types with counts, and code files on their own line.
2. Read `vaultpackage.xml` at the root. A complete manifest has a `name`, a `source` whose
   `author` is the Vault user name of whoever built the package, and a `summary`, plus
   `<javasdk><deployment_option>` when the package carries code. The `source` `vault` and the
   `description` may be blank. Fill in an empty `author` or `summary` and re-zip with the
   manifest at the root. An import that fails with "Invalid or missing inbound package manifest
   file" points back at this file.
3. Compare each packaged `.mdl` with what was meant to ship. The CLI assembles a component from
   the workspace's own files, so subcomponents written inline into a file added by hand can be
   missing from the package.
4. Look for dependencies it does not carry. A package that is missing a picklist the object
   needs will import fine and block on deploy.
5. For a Custom Page, check that the `Page` MDL's `page_controller` names a class under
   `javasdk/` and that its `client_distribution` names a folder under `websdk/`.

## Step 3: Deploy

The target is the Vault the user named, classified by login before this step. Against
`Production` or `UNKNOWN`, stop here: give the user the package path, the target DNS, the
classification, and the exact command below to run themselves.

```bash
MSYS_NO_PATHCONV=1 vault --verbose api "/objects/users/me" 2>&1 | grep -m1 '^> '   # must show the target DNS
vault config package deploy <package.vpk>
```

The command imports, deploys, and polls the deploy job. Never add `--yes`. When validation blocks the package, the command keeps polling and never reports the
block, so if it runs past a few minutes, stop it and read the package record (Step 4). Its poll
runs every ten seconds, and any job-status call made alongside it comes back
`API_LIMIT_EXCEEDED`.
The command has no `--dry-run`. The Vault's validation of the package is the validate call in
Step 5, or the validation log on the package record.

## Step 4: Read the result

A deploy is done when the Vault says so, not when the command returns. Check the package
record in Admin > Deployment > Inbound Packages, or through the API:

```bash
MSYS_NO_PATHCONV=1 vault api -H "Accept: application/json" \
  "/vobject/vault_package__v/<package_id>/actions/deploy/results"
```

The package record is the `vault_package__v` row whose `name__v` is the manifest's `name`;
`SELECT id, name__v FROM vault_package__v ORDER BY created_date__v DESC` finds it. Package
records stay on the Vault as history; an ordinary admin cannot delete them through the API.

The results list every step with its `deployment_status__v` and `deployment_action`. Report
the steps that did not land, by component name, and what blocked them. A package that
deployed some steps and failed the rest is not re-run blindly: the components that landed are
on the Vault now, and the user decides whether to fix the package or finish with ordered MDL
through the mdl skill.

## Step 5: Without the CLI

The same three moves, through the API, with `AUTH` and `VAULT_DNS` set the way login's
Step 5 prescribes, and the exact request shapes taken from the docs MCP. The import is a `PUT` to
`/api/{version}/services/package` with the `.vpk` as a multipart `file`; it returns a job, and
the job's result carries the new `vault_package__v` id. An import that comes back with an
empty body and no job id is worth sending once more before looking further. Validate before deploying with a
`POST` to `/api/{version}/services/vobject/vault_package__v/{package_id}/actions/validate`,
which says for every step whether each dependency is in the package, already in the target, or
missing. Stop on anything it marks blocked. The deploy is a `POST` to
`/api/{version}/vobject/vault_package__v/{package_id}/actions/deploy`; poll the job it
returns, then read the deploy results as in Step 4. Both jobs finish in seconds on a small
package, and Vault throttles job-status polls: a second poll a few seconds after the first
comes back `API_LIMIT_EXCEEDED`, "Too many polling requests". Wait well past ten seconds
between polls, or read the deploy results directly once the first poll says the job is done.

Without the CLI the package comes from the source Vault's export, or is built by hand. The
export runs through the API too, for an outbound package already set up in the source Vault.
Retrieve Outbound Package Dependencies
(`GET /api/{version}/vobjects/outbound_package__v/{package_id}/dependencies`) lists what it is
missing. A `package_component__v` record holding the `outbound_package__v` and
`vault_component__v` ids adds one; that is a write to the source Vault, so the production check
runs first. Export Package (`POST /api/{version}/services/package`, form field `packageName`)
starts the export job, and Vault sends the download link when it finishes.

## When it goes wrong

- Import succeeds and deploy blocks on a missing dependency: the package is short a component
  the target does not have. Add it to the package in the source Vault, or create it first
  through mdl, then deploy again.
- Java code fails validation: the import log names the class and the restriction. Hand it to
  java-sdk with the log line.
- The deploy command returned but the package record shows nothing: the import is still
  running. Wait on the job rather than deploying again.
