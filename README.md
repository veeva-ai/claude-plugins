# Veeva Claude Plugins

Claude Code plugins for working with Veeva Vault: signing in, querying, configuring, and building
on Vault. Install the plugin that matches your work and it brings the ones it depends on.

These plugins are new, and skills change between releases. Read the release notes when you update.

## Plugins

| Plugin | Install it if you are | Skills |
|---|---|---|
| [vault](plugins/vault/README.md) | Anyone working with a live Vault | login, query |
| [vault-administrator](plugins/vault-administrator/README.md) | An administrator reading, changing or moving configuration | config, file-staging, mdl, reporting |
| [vault-developer](plugins/vault-developer/README.md) | A developer using the Vault API, the Java SDK, Custom Pages or Vault AI Agents | api, java-sdk, custom-pages, ai-agents |

`vault-administrator` depends on `vault`, and `vault-developer` depends on both.

## Install

Add the marketplace once, then install the plugin for your role. Either line works; the first
needs only network access and Claude Code 2.1.224 or later, the second needs git on your machine.

```
claude plugin marketplace add https://download.veevavault.dev/claude-plugins/marketplace.json
```
```
claude plugin marketplace add veeva-ai/claude-plugins
```

Then:

```
claude plugin install vault-developer@veeva-claude-plugins
```

That brings `vault` and `vault-administrator` with it. Swap in `vault` or `vault-administrator`
if that fits your work better. Start a new Claude Code session afterwards so the skills load.
`claude plugin list` shows the version you have, for example `26.42.0`.

## Update

Plugins update when you ask, or on their own if you turn on auto-update for the marketplace in
`/plugin`:

```
claude plugin marketplace update veeva-claude-plugins
claude plugin update vault-developer@veeva-claude-plugins
```

Each release is tagged with its version, for example `26.42.0`; the release notes on the portal
say what changed.

## Without git or a current Claude Code

The same release ships as `claude-plugins.zip` on the
[GitHub release](https://github.com/veeva-ai/claude-plugins/releases/latest), with a `.sha256`
beside it. Unzip
it into a folder you will keep, so that the folder contains this README, then:

```
claude plugin marketplace add "<path to that folder>"
claude plugin install vault-developer@veeva-claude-plugins
```

To update, unzip the next release over the same folder; the next session loads it.

## Before you start

- A Vault user with API access. The `vault` plugin's login skill walks you through signing in.
- The Vault CLI is optional and recommended; the plugins support 26.2.4 and later. The
  [vault plugin README](plugins/vault/README.md) has the install line for Windows, macOS and
  Linux.
- Each plugin registers the public Vault developer docs MCP server at `docs.veevavault.dev`,
  which the skills use to look things up. It needs network access to that host.
- Try changes on a sandbox first. The skills run writes only against a Vault the login skill has
  classified as non-production. On production they hand you the command to run yourself.

## About this repository

It is published from an internal source on each release, so pull requests are not merged here.
Documentation and release notes live on the Vault developer portal.

## License

[Apache License 2.0](LICENSE)
