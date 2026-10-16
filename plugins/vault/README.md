# vault

Vault skills for any user connected to a live Vault: signing in, and querying records and
documents with VQL.

## Skills

| Skill | What it does | Example ask |
|---|---|---|
| [login](skills/login/SKILL.md) | Signs in to a named Vault with an API access token, a username and password, or an SSO token, stays signed in across tool calls, points every command at one Vault by its DNS, and classifies the target as production or not before anything writes. The base every other Vault skill in this marketplace calls. | "Sign in to my QA sandbox" |
| [query](skills/query/SKILL.md) | Turns plain-English questions into correct VQL, reviews existing queries for syntax errors and common failure modes, and runs confirmed queries against the connected Vault. | "Get all product records where status is active" |

## Setup

- Ships `.mcp.json` registering the public Vault developer docs MCP (`vault-docs-mcp`,
  auto-discovered on install). Needs network access to `docs.veevavault.dev`.
- The Vault CLI (`vault`) is the preferred way in: it signs in, stores credentials, and runs
  queries, MDL, and configuration pulls and pushes. It is not required. Without it the skills
  call the Vault API directly, using an API access token you export in your own terminal.
  `reference/vault-cli.md` lists what the skills assume about it. Minimum version is 26.2.4 build 631. On
  Windows, in PowerShell, run `irm https://download.veevavault.dev/cli/install.ps1 | iex`; it
  installs to `%LOCALAPPDATA%\VeevaCLI`, adds it to your user PATH, and the window you installed
  from can run it right away; other open terminals, and a Claude Code session already running,
  need a restart. On macOS or Linux, run
  `curl -fsSL https://download.veevavault.dev/cli/install.sh | sh`; it installs to
  `/usr/local/bin` when that's writable, otherwise `~/.local/bin`, and warns if that folder
  isn't on your PATH. After that, `vault upgrade` moves it to the latest release. Check with
  `vault --version`; if it reports something older than 26.2.4, another `vault` earlier on your
  PATH is answering first.
- query runs through the CLI's stored sign-in, or the exported token without it. Sign in
  before running queries.
- No `hooks.json`. Nothing runs at session start.

## Prerequisites

- A Vault user with API access. With the Vault CLI, SSO sign-in works in a browser. Without it,
  an API access token is the simplest credential; the login skill says where to generate one.
- On Windows, Git Bash or WSL for the API calls the skills make with `curl`. The Vault CLI
  itself runs in any shell.
- Supports Vault CLI 26.2.4 (build 631) and later. The CLI is optional; Setup above has the install line.
- query never prompts for credentials. It relies on the active sign-in.

## Handoffs to the other plugins

- Pulling configuration, File Staging transfers, and MDL changes are `vault-administrator`.
- Building REST calls beyond a VQL query is `vault-developer`'s `api` skill, which signs in
  through `login` here.
