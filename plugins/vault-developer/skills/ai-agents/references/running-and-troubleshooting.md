# Running and troubleshooting an agent

## Where users reach the agent

| Entry | Works when |
|---|---|
| AI Chat icon on a record page | The record is of the agent's object and an action is configured for that object; Vault AI Chat cannot be opened on a record whose object has no action configured |
| AI Chat icon on a document | The agent is document-scoped, and the icon shows on the document's Doc Info page |
| AI tab | Only for early adopters, and the user has Tabs: AI: View and Pages: AI: View |

For an object-scoped agent, the hand-off names a record of that object to open, not the AI tab.

## When an action fails or answers wrongly

Work down this list and stop at the first one that shows the cause.

1. **Agent Traces.** Admin > Logs > Agent Traces. Create a trace for the user who is chatting, run
   the action again, and download the JSON. It holds every LLM request, the tool specs as the model
   saw them, each `TOOL_CALL` with its arguments, each `TOOL_RESULT` payload, and `executionError`.
   If an execution is missing from the export, `agent_action_execution__sys` is the inventory and
   the trace the detail.
2. **SDK debug log on the chat user.** A custom handler runs for the user in the chat, so start the
   debug log session on that user. When the runtime log or a debug log on the agent's service user
   shows nothing for a custom handler, move the session to the chat user.
3. **Service-call counts.** When the debug log shows a handler start and end but the action still
   fails, compare the `SYSINFO` service-call counts (`JsonObjectBuilder` and the others) with what
   the code has to create. A short count points at a throw inside a lambda that the log does not
   print. Check this before calling the failure a platform problem.
4. **Execution status.** `GET /services/ai/agent_actions/status` answers only for an action with
   `api_access(true)`. Without it the call returns `INVALID_DATA`. Set `api_access(true)` while
   developing. Per the Agent MDL page it also makes the action callable from the Vault API and
   the MCP server, so turn it off again when that is not wanted.

To run an action by API, `POST /services/ai/agent_actions/actions/execute` with a JSON body of
`agent_name` and `action_name`. An object-scoped agent also takes `object`, `objecttype`, and
`record_id`; a document-scoped one takes `document_id`. The response carries an `executionId` and
a status URL to poll for the output.

Keep diagnostics beside the workspace, not inside it; `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`
says why.

An agent that has never run fails on import instead; see "Before anything is deployed" in `SKILL.md`.
