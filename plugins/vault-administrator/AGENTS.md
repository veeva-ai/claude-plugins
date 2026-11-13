# Vault Administrator - which skill

This plugin's skills read and change a Vault's configuration. The words a user picks decide the
skill, and two of them collide with the vault plugin's query skill, so the routing is written down
here rather than left to the descriptions.

## Which skill, before answering

| The user says | Skill | Why |
|---|---|---|
| report, dashboard, report type, chart, pie, tile, "a report of X by Y", "put that on a dashboard" | reporting | A Vault report is a component (report type, definition, layout) and a dashboard is a component over reports. The user wants something that exists in Reports afterwards, not rows on the screen |
| query, VQL, "get all records where", "pull all X", "how many", "list the", a one-off answer | vault -> query | The user wants data now, nothing saved in the Vault |
| field, object, picklist, lifecycle, "add a", "change the", a config script that failed | mdl | A single configuration change, written and run as MDL |
| "what is configured", "what depends on", "pull the config", a question answered from files | config | Reads the Vault's configuration into a workspace and answers from it |
| package, VPK, "move this to the other Vault", deploy | vpk | Configuration moving between Vaults as a package |
| process view, Process Monitor, cycle time, "how long from A to B", first time right, rework, variants on a lifecycle | process-monitor | A Process View is a component (`Processview` with a `settings` XML and charts) that Process Monitor runs. The user wants a view saved in Vault, not a one-off number |
| Vault AI metadata, synonyms for AI Chat, "the agent keeps confusing these objects", "how many entities are left" | ai-metadata | Drafts the AI metadata agents read (Vsm files and a review file for the SME); the mdl skill runs the reviewed files |
| upload, staging, "get the files up", File Staging | file-staging | Files in Vault File Staging before a load |

When a message carries both ("query the expenses and put them on a dashboard"), the reporting skill
owns it: a dashboard needs a report, and a report carries its own query. The query skill is the
right hand-off only for the one-off count the reporting skill uses to verify a build.

Cycle time asked for as a Vault report or a dashboard chart stays with the reporting skill. Once
the user says process view or Process Monitor, or wants time between two lifecycle states measured
over a timeframe, the process-monitor skill owns it.

## The target Vault

The Vault a skill works on is the one the user names. The CLI's current Vault, a stored
credential, or a workspace folder say what can be reached, not what should be. If the user has
not named the target, ask before any read that returns data from it and before any write.

## Writes

Every write to a Vault goes through the mdl, config, file-staging or vpk skill after the user's go on
the plan, with the target classified as production or sandbox first. The reporting skill drafts; it
does not execute.
