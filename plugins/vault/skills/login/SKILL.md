---
name: login
version: "0.1.0"
description: >
  For an admin, a developer, or another skill that needs a signed-in Vault. Use it to sign in
  to a named Vault with an API access token, a username and password, or an SSO token, stay
  signed in across tool calls, point every command at one Vault by its DNS, and classify the
  target as production or sandbox before anything writes.
  Also use it when a call fails with a login, token, or credentials error.
---

# Vault Login

The base skill for live Vault work. It answers four questions the other skills should never
answer on their own: which Vault am I pointed at, am I signed in, is it safe to write here,
and what kind of Vault is it.
The query skill here and the vault-administrator and vault-developer
plugins all call into this skill for those answers.

Sign-in and stored credentials belong to the Vault CLI (`vault`). Classifying the target and
holding writes back from production belong to this skill, with or without the CLI.
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` has the CLI's commands and the rules every
skill follows when using it.

## Scope

Signing in, staying signed in, signing out, and classifying the target Vault. Out of bounds:

- Writing the VQL, MDL, or REST call itself: query here, mdl in vault-administrator, api in
  vault-developer.
- Managing users and security policies: an admin task in the Vault UI, or api for the
  endpoint.
- Driving a browser or an OAuth flow. `vault login` opens the browser for SSO, and the user
  is the one who runs it.
- Finding an endpoint. Endpoints come from the docs MCP
  (`mcp__vault-docs-mcp__search_documentation`, source `vault_api_reference`) or from
  api, never from guessing a path or a field name. When the docs show no endpoint or
  field for a value, the answer is "not exposed through the API" plus the Admin page that
  holds it. An undocumented path that happens to answer is worse than a 404: it ends up in a
  customer's script and breaks without notice.

## Handoffs

When the ask is "query this", "load this", "run this MDL", or "call this endpoint", sign in
here, then hand the session to the skill that owns the work. Name it and stop. If a skill that
would do the work is not installed, say which plugin carries it (vault-developer for api)
and offer the drafted command instead of running it.

## Secrets stay out of chat

Never ask for a password, an access token, a session ID, or an OAuth token in chat, and never
run a command with one on its command line (`--password`, `--access-token`, `--session-id`).
The user runs sign-in in their own terminal. A secret pasted into chat is in the transcript
already: decline to use it and advise rotating it.

## Step 1: Check for the CLI

```bash
vault version
```

An answer at or above 26.2.4 means Steps 2 and 3. No answer, or an
older version, means offer to install or update it first: rule 3 in `reference/vault-cli.md` has the install line per OS,
the path for an older install, and the install-path fallback when this session's PATH has not
refreshed. Run the install only on the user's yes. If they say no or it fails, Step 5 for the
session and do not raise it again. Step 4 applies either way.

## Step 2: Resolve the target

Every `vault` command carries `--dns <dns>`, so the CLI's active environment never decides
where it lands. When the user names a Vault ("use my QA sandbox", "switch to prod"), the CLI has
no command that lists the Vaults stored on this machine. Take the DNS from what they said, say
which DNS you are using, and ask for it when they gave only a name: an invalid DNS can
authenticate against the Vault the user last logged into.

To see which Vault the CLI points at when `--dns` is left off, make the domain call without it:

```bash
MSYS_NO_PATHCONV=1 vault api "/objects/domain?include_application=true"
```

The response names the domain (`domain_name__v`) and each Vault in it (`vault_name__v`). The
domain name is part of the DNS of every Vault in it, so a domain with several Vaults still needs
the user to say which one is theirs.

### Two Vaults in one task

Copying configuration or moving data means a source and a target in the same session. Settle
both before the first command: the DNS of each, which one is the source, and the
classification of the target (Step 4). Say it back in one line, for example "source DEV
(mycompany-dev.veevavault.com, Sandbox), target QA (mycompany-qa.veevavault.com, Sandbox)".

From then on every `vault` command carries an explicit `--dns`. Never lean on the active
environment when two Vaults are in play: it is whichever one the user signed in to last, and a
push that lands on the source is the mistake this rule exists to prevent. Reads go to either
Vault. Writes go only to the one the user named as the target, and a write aimed at the source
stops for confirmation even when the source is a sandbox. The user names both Vaults; the Vault the CLI points at by default is not the target until they say so.

No call returns every Vault a user can reach by DNS. `GET /objects/domain` lists the Vaults in
the current Vault's domain by name and ID, and `GET /objects/domains` lists the user's domains
by name and type. Use them to narrow down which Vault the user means, then ask for its DNS.

Not signed in to that Vault, or the session has expired (a command with `--dns` ends in a
sign-in error): ask how they sign in to that Vault before
handing over a command. The bare `vault login` assumes SSO, and for an account with a Vault
password and no identity provider the browser flow just ends in "SSO login cancelled". One
question settles it: SSO, a Vault username and password, or an API access token they already
have. Then hand over the line that fits and wait for them to run it in their own terminal.

SSO:

```bash
vault login --dns mycompany-qa.veevavault.com
```

A username and password, or a token. `--password` and `--access-token` take no value: the CLI
asks for the secret with hidden typing, so it stays out of chat and out of their shell history.
`--password` needs `--username` with it. The same line works in PowerShell and bash:

```bash
vault login --dns mycompany-qa.veevavault.com --username me@mycompany.com --password
```

For a token:

```bash
vault login --dns mycompany-qa.veevavault.com --access-token
```

The CLI also reads the secret from one line of stdin when it is piped in. These lines are for
the user to run. Never run them yourself, and never ask for the value.

A password or SSO sign-in stores a session, and a session idles out, often in the middle of a
long task: the next command fails with `INVALID_SESSION_ID` and the user has to sign in again.
The CLI keeps no password, so it cannot do that for them. Right after a password or SSO
sign-in, offer an API access token so the rest of the work does not idle out. The user creates
it in the API Access Tokens section of their user profile in Vault, which shows the value once,
and signs in with it in their own terminal:

```bash
vault login --dns <dns> --access-token
```

`--access-token` takes no value on the command line: the CLI reads the token from a no-echo
prompt, or from one line of stdin. The line is the user's to run, never yours.

Recommend the credential that fits them rather than taking whichever comes first: an API
access token is the first choice, it is the only option besides SSO for a SAML user, and it
does not expire with an idle session.

Anyone who signed in through the older `vault_login.sh` helper signs in again with
`vault login`. The two never shared a credential store.

`vault logout <dns>` signs out of one Vault and ends that session in Vault; with no DNS it signs
out of the active one.

## Step 3: Stay signed in across calls

`vault query`, `vault mdl`, and `vault config` read the stored credential themselves. Nothing
to export and nothing to refresh; pass `--dns` and run the command.

A call the CLI has no command for (Vault File Staging, a load, a job, any other endpoint)
goes through `vault api`, which sends the stored session itself. No variable holds a
credential:

```bash
MSYS_NO_PATHCONV=1 vault api --dns mycompany-qa.veevavault.com "/objects/users/me"
```

(`MSYS_NO_PATHCONV=1`: see `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`.)

Do not read the credential out of the CLI's store to build a request
by hand. The CLI already holds the session and the client ID, and a second hand-built client
means two places for a credential to leak and two places for a header to be wrong.

State the auth mode in play (access token or session) when handing the session to another
skill, so it reads the header correctly.

## Step 4: Classify and profile before writing

Before any `POST`, `PUT`, or `DELETE`, a load, an MDL execute, a `vault mdl`, a
`vault config push`, a package deploy, or a job start, read the domain. With the CLI, let
`vault api` make the call: it signs in with the stored credential, so there is no header to
build and no secret in the transcript.

```bash
export MSYS_NO_PATHCONV=1
vault api "/objects/domain?include_application=true" --dns mycompany-qa.veevavault.com | grep -o -E '"(domain_type__v|vault_name__v|name__v|name)" *: *"[^"]*"'
```

`vault api` takes a path that starts with `/` and adds the `/api/<version>` segment itself, so
leave that off (the `export` is for Git Bash; see `vault-cli.md`). Without the CLI, use `AUTH`
and `VAULT_DNS` from Step 5 and call the same path with `curl`:

```bash
curl -s -H "Authorization: $AUTH" -H "Accept: application/json" \
  "https://$VAULT_DNS/api/v26.2/objects/domain?include_application=true" \
  | grep -o -E '"(domain_type__v|vault_name__v|name__v|name)" *: *"[^"]*"'
```

`domain_type__v` is `Production`, `Sandbox`, `Demo`, or `Test`. The API reference says the
call is for Domain Admins; in practice a Vault Owner who is not one gets the full response.
If it does fail on permissions, `GET /objects/domains` is the fallback: it lists
every domain the user belongs to with a `type`, so believe it only when every row has the
same type. Anything else, a failed call included, is `UNKNOWN`, and `UNKNOWN` counts as
production. Take the current API version from the docs MCP if the version in the curl call is
rejected.

The same response profiles the Vault, which is why the call asks for applications. Each
Vault in the domain comes back with its `vault_family__v` (`name__v` such as `quality__v`)
and its `vault_application__v` list. A domain with one Vault is unambiguous. A customer
sandbox domain can hold dozens; match on the name the user calls it or ask which row is
theirs, and do not guess from the DNS.
`GET /api/` (no version in the path; through the CLI, `vault api --api-version none "/api/"`) lists every API version the Vault answers. The last
entry can be the beta of the next release, so the release is the last entry the docs MCP
knows as general; on a Vault that lists v26.2 and v26.3, say v26.2 and mention v26.3 exists.

Say all of it on the one line this skill already states before a write: target DNS,
classification, family, release. "DEV (mycompany-dev.veevavault.com), Sandbox, Platform, API v26.2"
and then never ask again in the session. The family and the applications do not change, so
keep them in a note per DNS for next time; the classification is read live before every
write regardless, and the version is re-read when a call is rejected for it. If the domain
call failed, the family is `UNKNOWN` too. Say so, and do not infer it from custom objects.

The profile shapes the rest of the work. A feature the docs put in a later release than the
Vault's API version is "not on this Vault yet", not a suggestion. Standard objects of
another family are not assumed present. When the profile is first established, name once the
plugin here that covers what the user came to do on this Vault, and leave installs alone
after that. On both paths the call above is the source of this profile.

The CLI does not make this check, and a command that runs is not evidence the target is safe.
The rule every skill in these plugins follows: reads run anywhere. Writes run against a Vault
classified as non-production. Against `Production` or `UNKNOWN`, do not execute; show the
target DNS, the classification, and the exact command, and hand it to the user to run. State
the target and classification in one line before any write, even on a sandbox, so the blast
radius is visible before it goes out. Never add `--yes` to get past a prompt.

DNS names are not a substitute for this check: a sandbox can live on a plain `.veevavault.com`
host.

## Step 5: Without the CLI

There is no credential store to read, so the session comes from the user's own shell. Ask
them to set two variables in the terminal they launch Claude Code from, not in chat:

```bash
export VAULT_DNS=mycompany-qa.veevavault.com
export VAULT_ACCESS_TOKEN=veeva-vault-...
```

In PowerShell:

```powershell
$env:VAULT_DNS = "mycompany-qa.veevavault.com"
$env:VAULT_ACCESS_TOKEN = "veeva-vault-..."
```

Then `AUTH="Bearer $VAULT_ACCESS_TOKEN"` and `export MSYS_NO_PATHCONV=1` at the top of each
block, and Step 4 applies unchanged. An API access token is the credential to
ask for here: it needs no login call, so
there is no session to cache or refresh. Check that both are set with
`[ -n "$VAULT_ACCESS_TOKEN" ] && echo set`, never by printing them.

A user who can only sign in with a password needs a login call first. Take the auth endpoint
and its parameters from the docs MCP, have them put the password in an environment variable
the same way, and keep the returned session ID in a shell variable for the rest of that
block. Do not retry a failed login: failures count against the auth burst limit, which is
counted per username and Vault DNS over a one-minute window.

## Step 6: When a call fails to authenticate

- `INVALID_SESSION_ID`: an expired or ended session, or a disabled or revoked API token. With
  the CLI, the fix is the user running `vault login --dns <dns>` again. With a token, the
  token itself is the problem.
- A token/username mismatch, or a credential Vault rejects: stop and say so. Never fall back
  to a different credential than the one the user chose.
- Repeated auth failures then a lockout-shaped error: the auth burst limit. Wait it out rather
  than retrying.
- A CLI command that exits with a sign-in error while the user says they are signed in: check
  the DNS you passed against the one they meant, and pass `--dns` on every command.

`references/auth-and-errors.md` has the credential types, session duration, the response
envelope, rate limits, and the tracking headers to capture for Support.
