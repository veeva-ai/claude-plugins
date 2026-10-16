# The Vault CLI in these plugins

The Vault CLI (`vault`) is a Veeva product. The skills drive it and do not reimplement it.
This file is everything the skills assume about it, so a CLI change is an edit here and
nowhere else. The copies under the other plugins' `reference/`
folders are identical to this one; edit this file and run `scripts/validate-all.sh`.

## The rules

1. Check once per task: `vault version`. No answer, or a version older than 26.2.4, means the
   CLI is missing for this task.
2. CLI present and the operation is in the table below: the run step is that command. No curl
   and no script for something the CLI does.
3. CLI missing: offer to install it before touching the Vault, in one short message, and run
   the README's install line for the user's OS once they say yes (the permission prompt is the
   yes; never run it unasked). Windows PowerShell:
   `irm https://download.veevavault.dev/cli/install.ps1 | iex`; macOS or Linux:
   `curl -fsSL https://download.veevavault.dev/cli/install.sh | sh`. A session that was already
   running may not see the new PATH, so if plain `vault` still answers nothing, call it by its
   install path: `%LOCALAPPDATA%\VeevaCLI\vault.exe`, or `/usr/local/bin/vault` and then
   `~/.local/bin/vault`. If `vault version` still reports the previous version after the
   install, another `vault` earlier on PATH is answering first; say which one.
   An installed CLI older than 26.2.4: `vault upgrade --check` shows the newer release and
   installs nothing. `vault upgrade` itself asks for confirmation, which a shell without a
   terminal cannot give, so hand it to the user to run in their own terminal, or run the install
   line on their yes.
   The user says no, or the install fails: carry on. Take the request shape from the Vault docs
   MCP (`mcp__vault-docs-mcp__search_documentation`) and call the API, and do not raise the
   install again.
4. Before any write, on either path, the production check from the vault plugin's login skill
   runs first. The CLI does not know whether a target is production. Reads skip the check.
5. Never pass `--yes` (`-y`), and never suggest it. Every command accepts it, and it skips both the
   confirmation prompt and the CLI's safety checks. One exception: when `vault config pull`
   stops before removing files the Vault does not have, and you committed the workspace just
   before, rerun that same pull with `--yes`, restore your files from git, and say you did.
   Never on a push, never as a first move, never to pull a different Vault into a workspace.
6. After a write, read back what changed. `vault mdl` prints "MDL executed." and nothing else,
   so its output says the call was accepted, not that the component changed. A read-back that
   comes back as an HTML page never reached the API (under Git Bash, `MSYS_NO_PATHCONV=1` was
   missing); fix the call and read again rather than leaning on the push report.
7. Never put a secret on a command line you run: `--password`, `--access-token`,
   `--session-id`. Sign-in is the user's to run.

8. A command handed to the user has to work in the user's shell, so ask which one they use
   before handing over the first. In Windows PowerShell, `~` is not expanded for a program's
   arguments (use `"$HOME\.veeva-vault\..."`), `&&` does not exist (one command per line), and
   a bash script runs as `& "C:\Program Files\Git\bin\bash.exe" "<path to script>"`. A script
   that pauses for input runs in the user's own terminal, not through Claude Code's `!` prompt,
   which has no keyboard attached.

## Commands

`--dns <dns>` picks the Vault on every command below; without it the CLI uses the
active environment, the Vault the user signed in to last. Pass `--dns` on every command so the
active one never matters.

| Operation | Command |
|---|---|
| Is the CLI here | `vault version` |
| Sign in (SSO in a browser by default) | `vault login`, or `vault login --dns <dns>` to skip the Vault picker |
| Which Vault the CLI points at | `vault api "/objects/domain?include_application=true"` with no `--dns`; the response names the domain and the Vaults in it. No command lists the stored environments |
| Vaults in the user's domain | `vault api "/objects/domain?include_application=true"` returns each Vault's name and ID, no DNS. `vault api "/objects/domains"` returns the user's domains by name and type. Neither returns a Vault DNS, so ask the user for the DNS |
| Sign out | `vault logout <dns>`, or `vault logout --all` for every stored environment |
| Sign in with an API access token | `vault login --dns <dns> --access-token`. The token is read from a no-echo prompt or one line of stdin, never from the command line. Create the token in the API Access Tokens section of the user's profile in Vault |
| Run VQL | `vault query "<vql>"`, with `--format table\|json\|csv`, `--all` for every page, `--page <n>` for one page. A page holds 1000 records. It has no output-file flag; redirect to save (`> rows.csv`) |
| Execute one MDL statement | `vault mdl "<statement>"` |
| Pull configuration | `vault config pull --dir <dir>`, with `--dry-run` |
| Preview a push | `vault config push --dir <dir> --dry-run` prints the plan and deploys nothing |
| Deploy the workspace | `vault config push --dir <dir>`. A push that would delete components lists them and stops for confirmation, which is the user's call |
| Call any REST endpoint | `vault api "<path>"` with a leading slash, and `-X`, `-d`, `-F`, `-H`, `-o <file>`. It adds the session itself |
| Read a component back as MDL | `MSYS_NO_PATHCONV=1 vault api --api-version none "/api/mdl/components/<Type>.<name>" -o <file>`, then read the file. The MDL endpoint is unversioned and returns plain text, so it needs both flags. Give `<file>` as a relative path: on Windows, `vault` writes `/tmp/x` to `C:\tmp\x`, where Git Bash does not look |
| Run VQL in a form body | `vault api -X POST -H "Content-Type: application/x-www-form-urlencoded" -d "q=<VQL>" /query`. Write a `LIKE` wildcard as `%25`; a bare `%` comes back as "Cannot parse the query [null]" |
| Build or import a VPK | `vault config package create --dir <dir> --output <file>.vpk` (changed components only), `vault config package deploy --dns <dns> <vpk>` |

`vault mdl` takes the whole statement as one argument. MDL uses single quotes for strings,
so wrap the statement in double quotes, and escape any double quote inside it.

The loop for a configuration change in a workspace, new components included, is `pull`, edit
the files, `push --dry-run`, `push`, then `push --dry-run` again. The second preview should
report no changes. The preview compares the files with the Vault as it is now, so in a workspace
that is behind, a change made in Vault since the last pull shows up as an update the push would
undo, and a component added in Vault since shows up as a delete. The pull comes first every time.

Keep the workspace in git and commit before every pull. A pull replaces the files Vault
owns, and the commit is what makes it safe to try one.

If `cli-interim.md` exists beside this file, read it before a pull or a push. It
lists what to do when a command misbehaves.

## No command yet

These take the API path: Vault File Staging, job status, and any endpoint that is not in the
table. Call them with `vault api "<path>"`, which adds the session itself. Data loads go
through the vault-developer plugin's api skill, its Step 6, and no skill submits one any other
way. Never read the session out of the CLI's credential store to build a request by
hand. The CLI already holds the session and the client ID, and a second hand-built client
means two places for a credential to leak and two places for a header to be wrong.
Under Windows Git Bash, set `MSYS_NO_PATHCONV=1` on any command with an argument that starts
with `/`, `vault api "/..."` included. Git Bash rewrites such an argument into a Windows path,
so a staging path like `/u1234/loads` reaches Vault as `C:/Program Files/Git/u1234/loads` and
the call fails. Elsewhere the setting does nothing. Windows
has no `zip` command in Git Bash; build an archive with PowerShell's `Compress-Archive`.

Without the CLI the user exports `VAULT_DNS` and `VAULT_ACCESS_TOKEN` in their own terminal,
and the block is `AUTH="Bearer $VAULT_ACCESS_TOKEN"`. The vault plugin's login skill owns both
paths and the production check that comes before a write.

## Workspace layout

```
<workspace>/
  components/<Type>/<name>/<name>.mdl             the component
  components/<Type>/<name>/<Sub>/<subname>/       one folder per subcomponent: its .mdl,
                                                  and its XML sidecar when it has one
  javasdk/                      Java SDK source by package path, com/veeva/vault/custom/...
  custom_pages/<name>/          the built Custom Page distribution and its manifest
  .vault/                       CLI state: file hashes, the dependency graph, environments
  .vdxignore                    globs the CLI leaves alone
```

A subcomponent is its own file, so an object's fields sit one per folder under
`components/Object/<name>/Field/`. Components whose body is XML keep it next to the `.mdl` and
name it from the `.mdl` (`rule(@rule.xml)`): `rule.xml` on a lifecycle user action,
`step_detail.xml` on a workflow step, `page_markup.xml` on a page layout. A list attribute keeps
the form the pull wrote it in. The workspace has no `compare` and no `pull --type`; a pull
takes the whole Vault.

A pull writes over a file you changed without asking. It stops only before removing files the
Vault does not have, and in a shell with no terminal it exits asking for `--yes`. The commit
before the pull is how your edits come back. Source for a Custom Page, with its `node_modules`, lives outside the workspace;
`custom_pages/<name>/` holds the built files and the manifest.

Debug log zips, extracted logs and trace exports go in a folder beside the workspace, never under
the pulled tree: a pull can drop them and a push preview can list them.

The CLI builds a class name from the path under `javasdk/`. A Maven layout
(`src/main/java/...`) under it produces the wrong class name.

## Known limits

- `vault config push` deploys the classes under `javasdk/` along with the components, but
  `vault config package create` leaves `javasdk/` out of the VPK. Code that has to travel in a
  package goes in the source Vault's outbound package export or a zip built by hand, deployed
  with `vault config package deploy` or through the API. Between Vaults, import the package with
  Import Package (`PUT /api/{version}/services/package`) and deploy it with Deploy Package
  (`POST /api/{version}/vobject/vault_package__v/{package_id}/actions/deploy`).
- `vault config push --dry-run` lists every local `.java` file as an update on every run.
- `vault config package deploy` imports, deploys, and polls the deploy job every ten seconds.
  When validation blocks the package it keeps polling and never reports the block, so stop it
  after a few minutes and read the package record. A job-status call made while it polls comes
  back `API_LIMIT_EXCEEDED`.
- `vault config package deploy` has no `--dry-run`. The Vault's package validation is a
  separate API call, not part of the CLI.
- `vault config package create` writes an empty source `vault` and `author` into
  `vaultpackage.xml` when the session is dead, and Vault rejects that package on
  import ("Invalid or missing inbound package manifest file"). Fill in `author` and re-zip;
  `vault` is optional in the manifest and can stay empty.
- A component file added to a workspace by hand is packaged without its inline subcomponents;
  the CLI rebuilds components from its sidecar layout. Check the packaged `.mdl`.
- A push can send an empty sidecar as `{}`, which Vault rejects for some attributes. Look at
  the component's sidecar files when a push fails on an attribute nobody edited.
- `vault config pull --dry-run` against the workspace's own source Vault prints counts of what a
  pull would change and writes nothing.
- `vault login --password` and `--access-token` take no value: the CLI asks for the secret
  with hidden typing, or reads one line of stdin. `--password` needs `--username` with it. The
  vault plugin's login skill has the lines, for the user to run.
  A bare `vault login` assumes SSO, and for an account with a Vault password it ends in "SSO
  login cancelled".

A CLI error that is not in this list is the CLI team's to fix. Capture the command, its
output, `vault version`, the Vault release, and the ID in brackets at the start of that run's
lines in the CLI log (a failed command prints the log's path; Vault received the same ID as
`X-VaultAPI-ReferenceID`). Hand those to the user, and finish the task on the API path.
