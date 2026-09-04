---
name: controllerai-workflows
description: Build, inspect, edit, test, recover, and publish Controller AI workflows through the cai CLI or typed MCP tools. Use when a user asks to create or change an automation, workflow, flow, or workflow release, or when another Controller AI skill needs the shared lifecycle and safety rules.
---

# Controller AI workflows

The `cai ...` examples are canonical grammar. Follow the transport rules and MCP map in `controllerai-start`; never mix transports within one task. Do not call private REST endpoints or invoke MCP routes outside the typed tools. When the selected transport lacks a capability, say so and name the UI path instead of bypassing it.

## Rules that break tasks when skipped

- Lock the effective workflow ID and exact flow ID; see Establish context.
- Repair the selected workflow and flow instead of creating replacements; see Build protocol.
- Keep workflow, flow, node, input, and output IDs distinct; see Lifecycle rules.
- Preserve contract fields in place and refresh their dependents; see Lifecycle rules.
- Use save, history, and restore for recovery instead of rebuilding; see Lifecycle rules.
- Treat validation counts as status; prove the requested rail and stop. See Lifecycle rules and Before you finish.

## Establish context

```bash
cai auth status --json
cai workflow list --json
cai use <workflowId>
cai workflow status --json
```

MCP: use `account_status`, then call `workflow_list` and `workflow_status` with explicit `workflowId`; there is no `cai use` or stored pin.

If no workflow exists, create it with its final name:

```bash
cai workflow create --name "<clear outcome-oriented name>" --json
```

Read `selected`, `effectiveWorkflowId`, and `pinSource`; creation may not win the pin. If unselected, remove a command flag, use `cai use <id>` for a stored pin, or clear inherited `CAI_WORKFLOW`, as `pinSource` directs, then re-check. `cai config reset` clears only stored pins. `cai doctor` verifies the effective pin and development head. On MCP, capture and pass the new `workflowId`. Create with the final name.

A new workflow already ships with one empty flow. Rename that flow and build in it — do not add a second flow for the first piece of work; only add flows for genuinely separate canvases. A stray empty "Flow 1" left behind is clutter the user has to clean up.

Use IDs returned by the selected transport. For account-wide lists, filter by exact ID or name and owning workflow; never choose the first or latest similar result. Read the selected object's ID, name, and owner once, reuse those IDs, and never rediscover the target by name. A missing projected create field is not failure: inspect the complete result and exact inventory before retrying. Never invent identifiers.

To orient inside a workflow, prefer one call over many:

```bash
cai workflow outline --json
```

`cai workflow state --json` is bounded; add `--flow <flowId>` for node and edge identities. Use `workflow document` only for complete opaque JSON. `workflow outline` shows flows, contracts, graph `deps`, triggers, and data inventory. Use `flow get` for props and expressions.

## Build protocol

1. Translate the request into inputs, steps, outputs, trigger, integrations, failure behavior, and publish intent. When the request says to store, save, log, or record something and names no external destination, that means a data type plus a create-record node (`controllerai-data`) — a flow output alone does not persist anything. When the user names the destination (a spreadsheet, a CRM, another app, a file), write there with that integration's node instead; add a platform record only when the user also wants a copy kept here.
2. Inspect the current workflow before editing: `cai workflow outline`, then `cai flow get <flowId>`. Repair those IDs; do not create replacements for a repair.
3. Create or select a flow. Define flow inputs and outputs before nodes that derive props from them.
4. Build in execution order. After each material change, read it back with the corresponding `get`, `props`, `status`, or `list` command.
5. Run `cai flow layout <flowId>` once per changed flow as the LAST step after a headless build or any batch of structural edits (adds, deletes, rewires) — add-time placement avoids overlap, but only layout produces clean left-to-right columns. An open builder adopts your layout automatically.
6. Test on the development rail. Debug every failed or suspicious node; a green top-level status alone is insufficient.
7. Re-read `cai workflow status`. Publish only when the user asked for a live change or clearly authorized deployment.

For multi-domain work, read [references/command-map.md](references/command-map.md) before editing.

## Route specialized work

Load the narrow skill when the task reaches that surface:

- Node discovery, initialization, connections, props, or wiring: `controllerai-node-config`
- Dynamic values or conditions: `controllerai-expressions`
- Router, If/Else, loops, joins, or subflows: `controllerai-flow-control`
- Test execution or debugging: `controllerai-runs`
- Scheduled, webhook, or integration events: `controllerai-triggers`
- Schemas or records: `controllerai-data`
- Ask AI or Agent nodes: `controllerai-ai-nodes`
- User-facing agents and their workflow tools: `controllerai-agents`

## Lifecycle rules

- All graph edits target development. Live remains unchanged until `cai workflow publish`.
- Email verification is not a publishing prerequisite. An unclaimed agent-created account may publish; email claiming only raises its monthly usage cap and enables account recovery.
- On the CLI, prefer `--json` for decisions and IDs; human tables are for presentation. MCP tools return typed JSON. If human output looks odd, re-run the same read with `--json` before acting.
- A workflow contains flows; a flow is one canvas. Do not confuse their IDs.
- For flow inputs, pass a human `--display` label and use the current binding from `flow get` (`jsBinding`) or `workflow outline` (`binding`). Rename an existing contract input or output in place; do not delete and recreate it. After any rename or deletion, re-read the flow and repair every expression and subflow mapping before validation or execution. `legacyBinding` is diagnostic-only.
- Name nodes after the data they produce. Downstream input bindings are seeded from that name when wired.
- Use `cai workflow save --name "<checkpoint>"` before a risky rewrite. Named saves and state renames work on every plan.

```bash
cai workflow history --json
cai workflow history --group <groupId> --json
cai workflow restore --state <stateId> --json
```

- **Use version history for recovery.** Each save is a state. History retention is unlimited on Free, Max, and custom plans and 90 days on Pro/Starter; restore eligibility is 7 days on Free, 90 on Pro/Starter, and unlimited on Max/custom. Expand groups to capture state IDs. Restore rolls back the definition only; records and external effects remain.
- **Validate after structural edits and before publish.** Fix every listed prop, wire, expression, type, subflow, branch, and trigger error. Publish refuses errors; use `--acknowledge-validation-errors` only after explicit approval of the listed issues. Flow deletion also removes its contract, graph, and triggers and breaks callers; validation shows each break.
- Treat validation warning and error counts as status only. Never write them or their messages into unrelated workflow properties.
- On a conflict, re-read current state and reapply intent. Do not replay stale IDs or whole documents.
- Never delete a workflow without the user's explicit intent; the CLI requires `--yes` but that is not a substitute for authorization. MCP: `workflow_delete` has no `yes` parameter, but still requires the same explicit intent. `cai workflow delete` tears down deployed triggers and soft-deletes the workflow row. Historical workflow/version data remains stored, but there is currently no restore command for the deleted row.
- **Check dependencies before shared-resource changes.** `cai deps` shows attached agents, workflow callers, and connections; `cai deps --account` maps the account. Deleting a workflow with agent dependents is refused server-side, and `--force` is a deprecated, inert compatibility flag. Detach draft tools with `cai agent remove-tool <agentId> --tool <toolId>`. If still LIVE, use `cai agent publish <agentId>` after explicit approval. A dependent agent publish or delete is its own live/destructive decision. Then retry deletion.
- **Use the account feed only for recorded history.** Filter `cai events` by type, time, workflow, and cursor. It omits deletions, settings, member removals, and labels. For an unrecorded past change, say it is not recorded and change nothing; names and current state are not proof.
- **Share real product links.** Relevant publish, development run, agent, connection, workflow get/status, and agent get outputs include `appUrl` when known. Share returned links; never guess one. MCP `connection_connect_url` returns its link immediately.
- Never silently weaken the request to make a test pass. Report an unsupported capability or ask about a real product decision.

## Before you finish

Before claiming success, verify all of the following:

```bash
cai workflow outline --json
cai workflow validate --json
cai workflow status --json
cai flow get <flowId> --json
cai run flow <flowId> --inputs <values.json> --json
```

MCP: pass the parsed values in `inputs` and set `waitSeconds` on `run_flow`.

Then confirm:

- **`cai workflow validate` reports `errors: 0`.** Every error is fixed, or explicitly reported to the user with what it breaks — never summarized away, never called cosmetic. A red canvas and a "done" claim cannot coexist.
- Required node connections and props are complete.
- Expressions compiled and their resolved configs are non-null.
- Expected branch or error paths were exercised where practical.
- Flow outputs contain the promised values.
- The reported live/dev state matches what the user asked for.
- Final outline inventory matches the request. Remove only scratch schemas, flows, nodes, and triggers created during abandoned attempts.
- After fixing requested blockers and testing each changed path once, stop. Do not redesign for non-blocking warnings.

State what was built, what was tested, whether it was published, and any user-owned connection or product choice still pending.
