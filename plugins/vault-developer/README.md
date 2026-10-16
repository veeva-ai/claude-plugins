# vault-developer

Vault skills for developers: the Vault API, Java SDK entry points, Custom Pages, and Vault AI
Agents.

`java-sdk`, `custom-pages`, and `ai-agents` draft code and commands for you to review and
run yourself.
`api` runs calls against the Vault you sign into through `vault`'s
`login` skill, reads anywhere and writes only where that skill classifies the Vault as
non-production.

## Skills

| Skill | What it does | Example ask |
|---|---|---|
| [api](skills/api/SKILL.md) | Finds the right Vault API endpoint for a job, shapes the request, runs it against a signed-in Vault, and reads the response: pagination, job polling, logs, `responseStatus` and error types, and the traps in each endpoint family. | "Pull the last SDK debug log for this user and tell me why the trigger failed" |
| [java-sdk](skills/java-sdk/SKILL.md) | Writes Vault Java SDK entry point classes: record triggers, actions, jobs, custom APIs, and the services they call. Covers picking the right entry point, bulk-safe context patterns, and the platform constraints (JDK allowlist, `VaultCollections`, `LogService`) that block deployment if missed. Record triggers are the worked example. | "Write a trigger that sets this field whenever a record is created" |
| [custom-pages](skills/custom-pages/SKILL.md) | Scaffolds the three pieces of a Vault Custom Page: the Page MDL component, the Java `PageController`, and the client bundle. Covers client-server data flow, the `ValueType` mapping, and Shadow DOM styling. React/esbuild is the worked example; any framework works. | "Scaffold a custom page that shows a report of field usage" |
| [ai-agents](skills/ai-agents/SKILL.md) | Scaffolds the pieces of a Vault AI Agent: the `Agent` MDL and its tool and context wiring, the AI Tool Type and Context Type runtime handlers, the result handler that writes an action's output back, and the record action or trigger that starts an instance. Covers the standard tool types, the context configuration envelope, and the checks that catch a silent misconfiguration. | "Scaffold an agent tool that looks up glossary terms and wire it to an action" |

## Setup

- Ships `.mcp.json` registering the public Vault developer docs MCP (`vault-docs-mcp`, auto-discovered on install). It needs network access to `docs.veevavault.dev`, so on VPN check that first if grounding suddenly stops working.
- Depends on `vault` and `vault-administrator` from the same marketplace; installing this
  plugin brings both along. `api` signs in through `vault`'s `login` skill, the skills hand VQL
  work to its `query` skill, and MDL and File Staging work to `vault-administrator` by name. `api` takes its session from `login`, which reads it
  from the Vault CLI or, without the CLI, from two variables the user sets, so run `login` once
  before the first call.
- No `hooks.json`.

## Prerequisites

- A Vault user with API access for `api`. An API access token is the simplest credential;
  `login` covers the setup, including where the credentials file lives
  (`~/.veeva-vault/credentials`, which is `%USERPROFILE%\.veeva-vault\credentials` on Windows).
- On Windows, Git Bash or WSL for the bash blocks `api` runs. They do not run in PowerShell
  or cmd.
- Supports Vault CLI 26.2.4 (build 631) and later. The CLI is optional; the `vault` plugin README has the
  install line.
- A Java project to put generated `.java` files in; the skills write source, you compile it, and
  `vault config push` deploys it within one Vault.
- For the Custom Pages worked example: Node.js and npm (the scaffold's install steps cover the
  rest).

## Handoffs

Both plugins below install alongside this one, so these land on an installed skill rather
than a name.

- `api` sends query text to `vault`'s `query` skill, MDL scripts to `vault-administrator`'s `mdl`,
  file transfers to `file-staging`, and runs an approved bulk load or extract as a Vault Loader
  job (Step 6).
- `java-sdk` stops at source on disk and sends deployment to
  `vault-administrator`'s `config` and `mdl` skills.
- `custom-pages` sends MDL execution to `vault-administrator`'s `mdl` skill, query construction
  to `vault`'s `query` skill, and deploying the page to `config`.
- `ai-agents` sends MDL authoring and execution to `mdl`, the Java SDK platform constraints
  to `java-sdk`, running or checking an agent action over the Vault API to `api`, and
  deploying to `config`.
