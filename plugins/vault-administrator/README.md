# vault-administrator

Vault skills for administrators. Reads run against the Vault you sign into through the
`vault` plugin's `login` skill; writes run only against a Vault that login has classified as
non-production, and are handed to you as drafted commands otherwise.

## Skills

| Skill | What it does | Example ask |
|---|---|---|
| [config](skills/config/SKILL.md) | Pulls a Vault's configuration into a local workspace once, tells you when it has gone stale, and answers "how is X set up" from the files so the other skills stop calling the API component by component. | "Pull my QA vault's config and show me the fields on study__c" |
| [file-staging](skills/file-staging/SKILL.md) | Uploads folder trees and single files to Vault File Staging (VFS), including resumable uploads for large files, and lists, downloads, deletes, moves, and verifies what is staged. | "Stage this folder of PDFs before the document load" |
| [mdl](skills/mdl/SKILL.md) | Reads, authors, and executes Metadata Definition Language (MDL) for configuration components: statement syntax, RECREATE vs. ALTER semantics, multi-type object rules, a shape checklist for any script it emits or is shown, and checksum verification that a change actually landed. | "Add a text field to my study object across both its types" |
| [reporting](skills/reporting/SKILL.md) | Explains what an existing report or dashboard shows from its pulled files, traces reports to their report types and dashboards to their reports, drafts a clone or a report type, and builds a report type, reports and a dashboard from templates for the mdl skill to run on your go. Running reports and building them visually stay in the Vault UI. | "What does the Document Views report filter on?" |

## Setup

- Ships `.mcp.json` registering the public Vault developer docs MCP (`vault-docs-mcp`,
  auto-discovered on install). Needs network access to `docs.veevavault.dev`.
- Depends on `vault` from the same marketplace; installing this plugin brings it along. Every
  skill here signs in through its `login` skill.
- The Vault CLI (`vault`) is the run step wherever it has a command: pulls, plans, pushes,
  MDL, packages. Without it the skills call the documented API instead, and config answers
  one question at a time with no workspace. `reference/vault-cli.md` is what the skills
  assume about the CLI.
- No `hooks.json`. Nothing runs at session start.

## Prerequisites

- A Vault user with API access, signed in through `vault`'s `login` skill; that skill says
  where to generate an API access token.
- On Windows, PowerShell or Git Bash. The skills ask which shell you use before handing over
  a command.
- Supports Vault CLI 26.2.4 (build 631) and later. The CLI is optional; the `vault` plugin README has the
  install line.
- A sandbox to try anything in before it touches a shared Vault.

## Handoffs to the other plugins

- Signing in and classifying the target is `vault`'s `login` skill; writing and running VQL is
  its `query` skill.
- Building REST calls beyond a VQL query is `vault-developer`'s `api` skill.
- `mdl` refuses to author the Java SDK source an MDL statement references and hands it to
  `vault-developer`'s `java-sdk` skill by name, however the ask is worded.
- Moving components and code between Vaults as a package goes through Admin > Deployment >
  Outbound and Inbound Packages, or Import Package and Deploy Package over the API.
  Bulk loads and extracts run through `vault-developer`'s `api` skill (Step 6).
