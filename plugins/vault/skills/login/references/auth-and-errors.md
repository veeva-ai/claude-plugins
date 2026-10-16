# Authentication, Errors, Pagination & Rate Limits

What this skill needs to sign in and read back an error correctly. For a full endpoint or
field beyond what's here, ask the docs MCP (`mcp__vault-docs-mcp__search_documentation`,
source `vault_api_reference`) rather than guessing.

On a live call the authorization value is `$AUTH`, set from the Vault CLI's credential store
or from an exported token; see SKILL.md Steps 3 and 5. The `{SESSION_ID}` placeholder in the
examples below is for the other case, a
command being handed to a human to run instead of executed by the skill. A real credential is
never written into a file, a script, or the transcript either way.

## Credential types

Two distinct families of credential exist. Mixing them up, sending a session ID with `Bearer`
when the developer meant to send an API Access Token, or vice versa, is the most common
mistake, so confirm which one is in play before drafting the auth call.

### 1. A genuine, tracked session (`sessionId`)

Minted by username + password (`POST /api/{version}/auth`, form-urlencoded `username`/
`password`, optional `vaultDNS` to override the URL's DNS) or OAuth 2.0/OIDC (`POST
https://login.veevavault.com/auth/oauth/session/{oauth_oidc_profile_id}`, header
`Authorization: Bearer {access_token}` where that token is the identity provider's, not
Vault's). Both return the same envelope: `sessionId` plus `vaultId`/`vaultIds[]`.

Username/password auth creates a "User Login" Login Audit History row; the session can be
ended early (`DELETE /api/{version}/session`) or kept alive without a real call (`POST
/api/{version}/keep-alive`). **Auth-defaulting** can silently route the login to a different
Vault than the one requested: always compare the response's `vaultId` (and `vaultIds[]`)
against what was intended before trusting the session. The fallback order when the requested
DNS is invalid for the user: (1) the Vault the user last logged into, (2) the oldest active
Vault the user belongs to, (3) failure if neither exists.

Sending it on later calls: Vault accepts a `sessionId` either **raw**
(`Authorization: {sessionId}`) or **with a `Bearer ` prefix** (`Authorization: Bearer
{sessionId}`); both work identically. Pick one form and stay consistent across an
integration's calls.

### 2. A static API Access Token

Not a session: no `/auth` call, no Login Audit History row. The user creates one in the API
Access Tokens section of their Vault user profile. The API can create one too
(`POST .../users/me/api_access_token__sys`), but the skill never calls it: the value comes back
in the response and would land in the transcript. **`token__sys` is shown only once, at
creation**; there's no way to retrieve the value again later, only the token's metadata.

Always begins with `veeva-vault-`. Always sent **with** the `Bearer ` prefix; there's no
documented raw-token form for it (unlike a `sessionId`, which accepts both). Scoped to a
single Vault, carries the permissions of the user who generated it, doesn't grant Vault UI
access (REST-only), up to 25 active tokens per user. It is the preferred credential for new
integrations, since it removes the `/auth` round trip and the session-caching problem
entirely. Tokens do not carry over automatically when creating a
sandbox from production or promoting pre-production to production; they must be regenerated
there.

**Package deploy needs a genuine tracked session.** A static API Access Token authenticates
package import and validate, but deploy fails with `INSUFFICIENT_ACCESS` ("Unable to create
concealed session token from existing token"), even for a Vault Owner. The identical deploy
succeeds with a `sessionId` used verbatim as the Authorization value. ~ Plan deploy work on a
genuine session from the start rather than on a token. Never downgrade the API version to
route around the error; it masks the credential problem instead of fixing it.

## Session duration and keep-alive

- A session stays active as long as some activity happens within the Vault's configured
  maximum inactive duration (Admin-set, varies per Vault). Hard ceiling: **48 hours**, not
  configurable, regardless of activity.
- `POST /api/{version}/keep-alive` refreshes the idle timer without making a "real" call.
  `DELETE /api/{version}/session` ends the current session early; it doesn't affect the
  user's other active sessions.
- Reuse a session across calls rather than re-authenticating per call: repeated `/auth` calls
  both flood the Login Audit History and burn the Auth API Burst Limit below.

## Response envelope

`responseStatus` is the field to branch on, always, not the HTTP status code. Vault returns
HTTP 200 for essentially every request it processes at all, including ones whose body reports
failure. Values: `SUCCESS`, `WARNING` (processed, but with warnings worth surfacing),
`FAILURE` (user error), `EXCEPTION` (system error).

On anything but `SUCCESS`, read `errors[]`: each entry has a `type` (stable per API version,
safe to branch application logic on) and a `message` (human-readable, can change between
calls, debugging use only). Some failures, notably auth failures, also carry a top-level
`errorType` (e.g. `AUTHENTICATION_FAILED`).

`INVALID_SESSION_ID` means different things in different auth modes; SKILL.md Step 4 has the
read.

**Batch/bulk endpoints wrap this one level deeper.** The outer `responseStatus` can read
`SUCCESS` while individual items inside `data[]` each carry their own `responseStatus` and
their own `errors`; a batch call succeeding at the outer level doesn't mean every record
inside it succeeded. Always iterate `data[]` and check each row's own status.

## Parsing responses with embedded control characters

For any endpoint that can return user- or LLM-generated text (AI Agent action outputs,
document rich-text or plain-text field reads, SDK Debug Log file bodies inside the
`/logs/code/debug` files ZIP), Vault sometimes embeds raw newlines and other control
characters directly inside a JSON string value, as literal bytes rather than a `\n` escape
sequence. Strict JSON parsers reject this outright, including `jq` and most languages' default
parsers, typically with something like `Invalid control character at: line 1 column N`. The
behavior is intermittent, the same endpoint sometimes returns properly-escaped bytes and
sometimes raw, so when drafting response-handling code for one of these endpoints, recommend a
non-strict parse as the fallback rather than assuming the failure means malformed output, e.g.
Python's `json.loads(raw, strict=False)`.

## Pagination

Two distinct shapes; confirm which one an endpoint uses rather than assuming. VQL/query-style
endpoints (`/query`, `/query/components`, audit trail, job monitors) carry `responseDetails`
with `pagesize`/`pageoffset`/`size`/`total` plus `next_page`/`previous_page` URLs, present only
when another page exists. Vault File Staging (VFS) listings use a different mechanism, a
`next_page` URL carrying an opaque `cursor` valid for 30 minutes; the VFS endpoints in the
docs MCP cover that listing behavior.

## Rate limits

- **Burst Limit**: total API calls per Vault, fixed 5-minute window. `X-VaultAPI-BurstLimit`
  and `X-VaultAPI-BurstLimitRemaining` report the window; `X-VaultAPI-ResponseDelay`
  (milliseconds) appears only on a throttled response.
- **Auth API Burst Limit**: `POST /api/{version}/auth` calls per `username` + `vaultDNS`,
  1-minute window. At 50% remaining, responses start delaying; once exhausted, further auth
  calls fail outright until the next window. Doesn't apply to OAuth/OIDC or SAML/SSO. The
  exact call ceiling is not published.
- Vault does not enforce daily limits; there are no `X-VaultAPI-DailyLimit*` headers. Other
  per-endpoint limits (job status polling, for one) are in the docs MCP.

Reuse a session or token across every call rather than re-authenticating per call, reach for a
bulk endpoint instead of looping single-record calls, and back off when
`X-VaultAPI-BurstLimitRemaining` gets low.

## Version pinning

Every endpoint needs a real `{version}` segment filled in, in the form `vXX.X`; never leave the
literal `{version}` token in a command presented as ready-to-run without flagging it as a
placeholder. `GET /api/` (itself versionless) lists every version the Vault currently
supports; the last one listed is Beta, everything before it is GA and frozen for good.

`~` The general-release boundary moves roughly three times a year (`YY.1`/`YY.2`/`YY.3`).
Confirm the current GA version against `GET /api/` or the Vault's own Release Notes before
assuming any specific version named in this file is still the newest GA.

## Tracking headers

- `X-VaultAPI-ClientID` (or `?client_id=`): identifies the integration in the API Usage Log.
  Alphanumeric plus `.` `_` `-`, up to 100 chars.
- `X-VaultAPI-ReferenceID`: a free-form per-call correlation token, echoed back in the
  response header and logged, for threading integration-side logs to Vault's own.
- `X-VaultAPI-ExecutionId`: comes back on every response, Vault-assigned; capture it whenever
  a call fails and Support involvement is possible.
