---
name: query
version: "0.1.0"
description: >
  For any Vault user who needs to retrieve data from Vault records or documents.
  Turns plain-English questions into correct, ready-to-run VQL; reviews existing queries for
  syntax errors, display-name misuse, IS NULL/IS NOT NULL usage, or wrong relationship syntax;
  answers general VQL questions about clause order, relationship traversal, and pagination; and
  offers to run any confirmed query against the connected Vault. Trigger on: "write me a query",
  "help me with VQL", "/query", "get all records where...", "find documents owned by...",
  "pull all X from Y", "how do I get all...", a pasted VQL query for review, questions about
  slow or misbehaving queries, or any plain-English Vault data retrieval request, even if the
  user never says "VQL". Do not use for Java SDK code (vault-developer → java-sdk), MDL/schema
  changes (vault-administrator → mdl), building, explaining or cloning a report or a dashboard,
  which is a Vault Report component and not a query (vault-administrator → reporting), or bulk data extracts to a
  file, which are a Vault Loader extract job (vault-developer → api).
---

# VQL Query

Queries retrieve data. They never change it. This skill writes, reviews, and runs VQL SELECT
queries against a connected Vault.

## Scope

| In scope | Hand off to |
|---|---|
| Writing new VQL from plain English |  |
| Reviewing and fixing existing VQL |  |
| General VQL syntax and mechanics questions |  |
| Running confirmed queries against Vault |  |
| Java SDK code that wraps or uses a query | `vault-developer` → `java-sdk` |
| Bulk data extracts to a file (a Vault Loader extract job, not a paged query) | `vault-developer` → `api` |
| Schema inspection (object/field definitions) | `vault-administrator` → `mdl`, or `config` if the Vault has been pulled |
| Which configuration components exist, and their checksums | here, through `vault_component__v` (`references/query-targets.md`) |
| Building, explaining or cloning a report type, report or dashboard | `vault-administrator` → `reporting`. Running a report stays in the Vault UI. A count or list of report records (`report__sys`) or who ran what is a data question and stays here |
| Credential management and sign-in | `vault` → `login` |

## Vault login

This skill reuses the session established by `vault` → `login` when it
needs to connect to Vault, to look up field names with `SHOW FIELDS` or to run a confirmed
query. It never prompts for credentials or initiates a sign-in on its own. If the skill needs
to connect and no session is active, direct the user to run `login` first and return.
For tasks that do not require a Vault connection (general VQL questions, writing or reviewing
a query from names the user provided), proceed without one.

---

## Step 1: Classify the request

Decide which shape this is before doing anything else: the right steps downstream depend on it.

- **Translate**, a plain-English description of data the user wants (e.g., "get all documents
  I own not in steady state", "find active product records"). Needs to become a VQL query.
- **Review**, the user pastes an existing query and wants it fixed, reviewed, or explained.
- **Explain**, a general question about VQL mechanics without a specific data request (e.g.,
  "how do relationship subqueries work", "what's the right clause order").
- **Run**, the query is already confirmed and the user wants results.

Some questions sound like data requests that VQL can't answer, and a query that runs anyway
answers a different question. Say so plainly and name the path that can:

- Who changed a field or a lifecycle state, and when, is audit history. The audit trail
  endpoints (`vault-developer` → `api`) or the Audit Trail action on the document or record
  return it; `modified_by__v` and `modified_date__v` only hold the latest change.
- Views and downloads of steady-state documents are the exception, because the Document Usage
  object records them (`references/sample-queries.md`).
- Money spent on AI tokens isn't stored in Vault. Token counts are, on Agent Action Execution
  records (`references/query-targets.md`), so offer those as what the Vault holds. For trends,
  Vault Help's "Monitoring Agents & LLM Token Usage" page also describes Daily Agent Activity,
  which Vault keeps.

If the shape is **Review**, get the exact query text and the exact error or unexpected result
upfront. A paraphrase of the error ("it said something about null") won't match the known
failure modes, and guessing wastes turns.

---

## Step 2 (Translate): Pin the target and field names

API names are non-negotiable: a guessed name produces a query that parses cleanly and returns
zero rows, silently. That's worse than a syntax failure because it looks correct.

Ask for the object or document type if not stated. Ask for field API names if not provided.

Standard fields end in `__v` (e.g. `name__v`, `status__v`). Custom fields end in `__c`.
System fields end in `__sys`. **Never guess a `__c` name.** Custom names are vault-specific
and there is no universal "what it's usually called."

System data has names too, and they aren't yours to invent. Workflows, workflow tasks, jobs,
reports, binder contents, and document relationships each have their own target in
`references/query-targets.md`; check there before deciding a question isn't queryable. A
target or field that isn't in the references doesn't go into the answer as if it were. When
connected, a one-row probe (`SELECT id FROM {target} PAGESIZE 1`) settles it; offline, mark
it unconfirmed.

The probe can also come back with "not a queryable object" for a target the docs do name. What
a Vault exposes depends on the Vault and on the user: a feature that is off, an edition that lacks
it, or a permission the user does not hold all give that answer, and it means not available here,
not that the target does not exist. Say which Vault refused it, do not swap in a guessed target,
and offer the API or Admin path that reaches the same data (`references/query-targets.md` names
it where there is one).

If the Vault is connected this session, use `SHOW FIELDS FROM {target}` or
`SHOW RELATIONSHIPS FROM {target}` to confirm names, with the user's go-ahead before running
anything. This is something you run, not a command list you hand back to the user.

If the Vault is not connected and the name is still unconfirmed, offer two concrete paths:

> "I need to confirm `<what's unconfirmed>` before writing this. It's not something I can
> safely guess. Would you like to (a) connect your Vault so I can look it up directly, or
> (b) share a Configuration Report if you have one handy?"

Present both options; don't pick for the user.

---

## Step 3 (Translate): Surface all assumptions before generating

Before writing the query, explicitly state:

- Which query target you're using and why
- Which field API names you're assuming and what evidence backs them
- How you're reading ambiguous parts of the request (e.g., "documents I own" → the owner
  role via `WHERE id IN (SELECT document_id FROM doc_roles__sysr WHERE role_name__sys =
  'owner__v' AND user__sys = '{user id}')`. `owner__v` is not a queryable document field, and
  `created_by__v` is the creator, who may no longer own the document. Owner is a standard
  role, so `owner__v` holds; custom roles are lifecycle-specific, and
  `SELECT role_name__sys, user__sys, group__sys FROM doc_role__sys WHERE document_id = {id}`
  shows what a given document actually carries.)
- Which reading "my documents" gets. Take the owner role, as above, say so in the answer, and
  offer the creator (`created_by__v`) as the other reading, since people use "my" for both.
- "My recent" or "my favorite" anything is the `RECENT` or `FAVORITES` option on the target
  (`SELECT id, name__v FROM RECENT documents`), which reads the user's own lists in the Vault
  UI. An owner filter sorted by date returns a different, larger set.
- Every filter the question states: a date window, a user, an activity type, a relationship
  type. Finding the right target is half the job, and a dropped filter quietly answers a
  broader question.
- Any picklist values used: if unconfirmed, use a `<CONFIRM_VALUE: description>` placeholder
  instead of a guessed literal, and name exactly what needs filling in

Silent assumptions produce queries that look right but answer the wrong question. Surfacing
them first lets the user catch mismatches before the query is written.

Then ask the user to confirm before generating.

---

## Step 4: Build the query

Check the reference files before writing. Don't rely on SQL intuition. VQL has specific
syntax and behaviors that diverge from SQL in exactly the places that produce subtle errors.

| Load when... | Reference |
|---|---|
| Always (clause order, SHOW, FIND, ORDER BY, pagination) | `references/clauses.md` |
| Query involves a relationship or subquery | `references/joins.md` |
| Building WHERE conditions (especially null checks) | `references/operators.md` |
| Using ALLVERSIONS, TOLABEL, TONAME, STEADYSTATE(), date literals | `references/functions-options.md` |
| Querying documents or need target-specific rules | `references/query-targets.md` |
| Building any query, check for an adaptable starting shape | `references/sample-queries.md` |

A sample's extra conditions belong to the question it was written for. When you adapt one,
keep the conditions the user's question asks for and drop the rest.

### Correctness rules

These rules catch the most common sources of wrong results or parse failures. They're not
arbitrary restrictions: each one exists because VQL diverges from SQL in a way that produces
silent failures.

**Clause order:** `SELECT` → `FROM` → `FIND` (optional) → `WHERE` (optional) →
`ORDER BY` (optional) → `MAXROWS`/`PAGESIZE`/`PAGEOFFSET` (optional; `MAXROWS` comes before
`PAGESIZE` when both appear). Inverting this order produces a parse error.

**No `SELECT *`:** List specific fields. This makes intent explicit and avoids fetching
unnecessary data.

**Labels vs. API names:** Document queries filter by label by default: `type__v =
'Promotional'` matches, and `type__v = 'promotional__c'` returns zero rows with no error.
Object queries filter by API name by default: `state__v = 'closed_state__c'` matches, and
`state__v = 'Closed'` returns zero rows with no error. `TONAME()` is for document fields and
compares against the API name; `TOLABEL()` is for object fields and compares against the
label. Don't use either on the other target; it's redundant there. Prefer
`TONAME(type__v) = 'promotional__c'` on documents, since it survives localized labels. Either
way the failure is silent: the wrong representation parses fine and returns nothing.

**`FROM documents`, not `FROM documents__v`:** Documents is a special query target. It
doesn't follow the `__v` suffix convention of object targets.

**Relationship subqueries use `__vr` tokens, not object API names:** The inner `FROM` in a
subquery must be the relationship token (e.g. `doc_roles__sysr`, `players__vr`), not the
related object's name (`player__v`). Using an object name where the token belongs often parses
fine and silently returns wrong data. See `references/joins.md` for the full distinction.

**Null checks: `= null` and `!= null` only.** `IS NULL` and `IS NOT NULL` are SQL, not VQL. Vault rejects them. Also note: `!= 'value'` in VQL includes rows where the field is null, so
add `AND field != null` when you need to exclude nulls explicitly.

**No leading wildcard in `LIKE`:** `LIKE 'prefix%'` only, on every target. A `SELECT` with
`LIKE '%term%'` is rejected. When the user wants a word anywhere in a name, use
`FIND ('term' SCOPE name__v)` on documents or a standard-volume object (`FIND` doesn't run on raw
objects); the scope takes the field's API name, and without it `FIND` also matches the other text
fields. Offer the prefix match only as the fallback, and say what it misses.

**`FROM documents` returns the latest version; `ALLVERSIONS` + `LATESTVERSION` finds the
latest version that matches a condition:** Plain `FROM documents` returns the latest version
of each document. `FROM ALLVERSIONS documents` returns every version. `SELECT LATESTVERSION
... FROM ALLVERSIONS documents` returns, per document, the latest version that meets the
`WHERE` or `FIND` criteria (e.g. the newest version where a field is still empty). See
`references/functions-options.md` for confirmed syntax.

**Picklist values must be confirmed, not guessed.** A plausible-looking picklist literal that
happens to be wrong silently returns zero rows, no error, no warning. Use a
`<CONFIRM_VALUE: description>` placeholder if the value is unconfirmed, and name what needs
filling in. Never present a flagged guess as if it were real VQL.

**No aggregates:** `COUNT()`, `SUM()`, `AVG()`, `GROUP BY`, and `HAVING` are not VQL; the
literal translation returns `INCORRECT_QUERY_SYNTAX_ERROR` or `function [X] is unknown`.
"How many" is the filtered `SELECT` with `PAGESIZE 0`, reading `responseDetails.total` off the
response; on the CLI, Step 6 shows how to print it. "Group by", "sum", and "distinct" are the
raw field selected with a page size that covers the population, then grouped, summed, or
deduped client-side.

---

## Step 5: Present the query and offer to run it

Present the query in a code block. Add 1–3 sentences on the target, filter logic, and join
choices, and call out any assumption made. Then offer the next step:

> "Ready to run this against your Vault? I can execute it once you confirm."

Include this offer on every response that contains a generated or reviewed query. Never
execute before the user explicitly confirms.

---

## Step 6: Run on explicit confirmation

When the user confirms, check for the Vault CLI with `vault version`. When it answers at or above
26.2.4, the CLI runs the query with the credential it already holds. No
answer, or an older version, means offer to install or update it first, per rule 3 in
`${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`, and use the API path only after a no or a
failed install.

```bash
vault query "<your VQL here>" --dns mycompany-qa.veevavault.com
```

Wrap the VQL in double quotes; its string values use single quotes. `--dns` is the DNS the
user named, and without it the query runs against the active environment, so
name the DNS you are about to hit. `--format json` or `--format csv` when the result feeds
something else, `--all` only when the user has asked for every page. A page holds 1000 records. A
sign-in error goes to login; do not ask for a credential here.

A count-only question needs the total, not the rows. The count is the filtered `SELECT` with
`PAGESIZE 0`, and the number is `responseDetails.total` in the query response, so run it through a
path that returns that field:

```sql
SELECT id FROM product__v WHERE state__v = 'active_state__v' PAGESIZE 0
```

Run one such query per count; do not page through rows to count them, since that spends a call
per page for a number the first response already holds. If
`${CLAUDE_PLUGIN_ROOT}/reference/cli-interim.md` exists, it says how to read the total through
the CLI. `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md` has the rest of the CLI's rules.

Without the CLI, post the query with the header login's Step 5 sets up:

```bash
AUTH="Bearer $VAULT_ACCESS_TOKEN"
curl -s "https://$VAULT_DNS/api/v26.2/query" \
  -H "Authorization: $AUTH" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Accept: application/json" \
  -H "X-VaultAPI-ClientID: claude-plugins" \
  --data-urlencode "q=<your VQL here>"
```

Always use `--data-urlencode`, not `-d`: query text contains `=`, spaces, and quotes that
`-d` won't escape correctly.

Two more things that turn a correct query into a syntax error: the query text is capped at
50,000 characters, which a generated `CONTAINS (...)` list of IDs can reach, and a single quote
inside a value must be escaped as `\'` or `''` (a literal `%`, `*`, or backslash as `\%`,
`\*`, `\\`).

Say which Vault the query ran against, by DNS. If the result includes
`next_page`, show the first page and the total, and ask before pulling more. `responseStatus`
can also be `WARNING`: the data is still attached, and it means the query was paged manually
with `PAGEOFFSET`, the same query string was repeated inside five minutes, or an unfiltered or
`ORDER BY` query ran long. Treat it as a rewrite signal, not a failure. If the user would
rather run it themselves, give them the same command with `{SESSION_ID}` and `{vault_dns}`
placeholders and stop.

---

## Review path

When the request is a Review:

1. Read the query as if building it from scratch. Identify the target, joins, filters,
   and functions used.

2. Check for these failure patterns (all covered in `references/`):
   - Labels where API names belong, or the reverse. Look for quoted strings that resemble
     human labels rather than API names with `__v`/`__c` suffixes, and vice versa. On an
     object field a bare label returns zero rows; the fix is `TOLABEL(state__v) = 'Closed'`
     if they mean the label, or the API name if they know it. On a document field a bare
     label is the default and works, so don't flag it as an error; suggest
     `TONAME(type__v) = 'promotional__c'`, which holds up across localized vaults, and flag a
     bare API name on a document field, which returns zero rows. Only fall back to a
     `<CONFIRM_VALUE>` placeholder if the user needs an API name nobody has confirmed.
   - `IS NULL` or `IS NOT NULL`, SQL syntax, not valid VQL
   - `SELECT *`, not allowed; specific fields must be listed
   - `FROM documents__v`, must be `FROM documents`
   - Object API names used as relationship tokens in subquery `FROM` clauses
   - `FIND` placed inside `WHERE` instead of as a top-level clause after `FROM`
   - A `LIKE` pattern that starts with `%`, which Vault rejects in a `SELECT`

3. For anything ambiguous (e.g., `FIND` and `WHERE` both present, valid, but unusual),
   flag it and confirm whether both were intentional.

4. Return the improved query in a code block with a concise explanation of each change and
   why it matters.

5. Include the run offer.

---

## Explain path

When the request is a general VQL question:

- Answer in prose before presenting any query.
- Any sample query must use placeholder names, `<object__v>`, `<field__v>`,
  `<relationship__vr>`, not real API names presented as if correct for the user's vault.
- Never present a runnable query with real API names in response to a "how does X work"
  question. Doing so implies those names are confirmed when they aren't.
