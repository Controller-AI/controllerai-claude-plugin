---
name: controllerai-workflows
description: Build, inspect, edit, test, recover, and publish Controller AI workflows through the cai CLI or typed MCP tools. Use when a user asks to create or change an automation, workflow, flow, or workflow release, or when another Controller AI skill needs the shared lifecycle and safety rules.
---

# Controller AI workflows

The `cai ...` examples are canonical grammar. Follow the transport rules and MCP map in `controllerai-start`; never mix transports within one task. Do not call private REST endpoints or invoke MCP routes outside the typed tools. When the selected transport lacks a capability, say so and name the UI path instead of bypassing it.

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

On the CLI, read `selected`, `effectiveWorkflowId`, and `pinSource` from the result; creation does not guarantee that the new workflow won the effective pin. If `selected` is false, rerun without `--workflow` when `pinSource` is `flag`; otherwise run `cai use <id>` for a stored pin or clear an inherited `CAI_WORKFLOW` pin in the parent environment, then re-check context. Use `cai config reset` to clear stale stored pins; it cannot clear an environment pin. `cai doctor` verifies that the effective workflow pin and its development head are readable. On MCP, capture the created `workflowId` and pass it explicitly thereafter. Do not create and then rename: `workflow create --name` submits the name directly.

A new workflow already ships with one empty flow. Rename that flow and build in it — do not add a second flow for the first piece of work; only add flows for genuinely separate canvases. A stray empty "Flow 1" left behind is clutter the user has to clean up.

Use IDs returned by the selected transport. Never invent workflow, flow, node, prop, event, field, route, or connection identifiers.

To orient inside a workflow, prefer one call over many:

```bash
cai workflow outline --json
```

It returns the whole map compactly — every flow with typed inputs/outputs, its nodes with upstream `deps` (the graph shape), triggers with source/enabled/armed state, and data types with per-rail record counts. Drill into a specific flow with `cai flow get <flowId>` only when you need props and expressions.

## Build protocol

1. Translate the request into inputs, steps, outputs, trigger, integrations, failure behavior, and publish intent. When the request says to store, save, log, or record something and names no external destination, that means a data type plus a create-record node (`controllerai-data`) — a flow output alone does not persist anything. When the user names the destination (a spreadsheet, a CRM, another app, a file), write there with that integration's node instead; add a platform record only when the user also wants a copy kept here.
2. Inspect the current workflow before editing: `cai workflow outline`, then `cai flow get <flowId>` for the flows you will touch.
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
- For flow inputs, pass a human `--display` label and use the compiler-current binding from the latest `cai flow get` (`jsBinding`) or `cai workflow outline` (`binding`). An input rename can change it: re-read bindings after every rename and never cache their spellings across renames. `legacyBinding` is diagnostic-only; never use it in expressions.
- Name nodes after the data they produce. Downstream input bindings are seeded from that name when wired.
- Use `cai workflow save --name "<checkpoint>"` before a risky rewrite. Named saves and state renames work on every plan.
- **Edit the graph with a plan-aware safety net.** Each save is a version state. `cai workflow history` has no age cutoff on Free, Max, or custom plans and a 90-day window on Pro/Starter; expand a group with `cai workflow history --group <groupId>` to get its state IDs. Restore with `cai workflow restore --state <stateId>` within 7 days on Free, 90 days on Pro/Starter, or without an age limit on Max/custom. Duplicate, delete, and refactor with that window in mind. Restore rolls back the DEFINITION (graph and schemas) only — record data is unversioned and stays as it is. External side effects (sent messages, provider calls) are equally outside the undo.
- **Validate after structural edits, don't ask permission before them.** After deleting a flow, deleting a type, or a large refactor — and before every publish — run `cai workflow validate` and fix what it lists. It is deep: required props, connections, wiring, expressions, output types, subflows, branches, trigger issues, grouped per red node. Publish refuses when validation reports errors; `--acknowledge-validation-errors` may only be used after the user explicitly approved publishing with the listed issues. Deleting a flow deletes its nodes, edges, inputs/outputs, and attached triggers, and breaks callers in other flows — exactly like deleting a function breaks its call sites; validate shows every break.
- On a conflict, re-read current state and reapply intent. Do not replay stale IDs or whole documents.
- Never delete a workflow without the user's explicit intent; the CLI requires `--yes` but that is not a substitute for authorization. MCP: `workflow_delete` has no `yes` parameter, but still requires the same explicit intent. `cai workflow delete` tears down deployed triggers and soft-deletes the workflow row. Historical workflow/version data remains stored, but there is currently no restore command for the deleted row.
- **Know the dependency graph before deleting or refactoring shared resources.** `cai deps` shows which agents attach this workflow as a tool (LIVE or draft), which workflows call it, and which connections it uses; `cai deps --account` maps the whole account. Deleting a workflow with agent dependents is refused server-side, and `--force` is a deprecated, inert compatibility flag. For a draft attachment, run `cai agent remove-tool <agentId> --tool <toolId>`. If it is also LIVE, publish that detached draft with `cai agent publish <agentId>`; if it is LIVE-only, publish the already-detached current draft or delete the agent. Publishing or deleting a dependent agent is its own live/destructive decision — get the user's explicit approval for that step first. Then retry the workflow delete.
- **Check the account feed when recent activity matters.** `cai events` lists what happened in the account, newest first: connections created, workflow edits/publishes/runs, agents created/published, members added. Filter with `--type` (comma-separated), `--since 30m|2h|7d|<ISO>`, `--for-workflow <id>`; page with `--cursor`. Typical uses: `cai events --type connection.created --since 10m` to see whether the user finished connecting, `cai events --since 24h` to catch up at session start, `cai events --for-workflow <id> --type workflow.run` for recent run activity. The feed cannot show connection deletions, settings changes, member removals, or any label/folder activity — absence there proves nothing. When the user asks you to reconstruct a past change the platform does not record (e.g. which workflow a label was removed from), say it is not recorded and change nothing; names or current state that merely suggest an answer are not evidence, and acting on a guess is worse than reporting the limitation.
- **Share the platform, not just text.** At the right moments, hand the user the real product page — command outputs include `appUrl` when the app origin is known: after `workflow publish` (the live builder view), after development `run flow`/`run node` (the run page; live flow runs omit it), and after `agent publish`/`test` (agent and conversation pages). On the CLI, `connection connect` prints the link; share it with `--no-open` before optionally adding `--wait`. MCP: `connection_connect_url` returns the link immediately and has no `open` or `wait` parameter. When the user asks for a link at any other time, don't guess URLs: `cai workflow get`, `cai workflow status`, and `cai agent get` include `appUrl`. When you cannot perform something (settings, product-only surfaces), give the link instead of apologizing.
- Never silently weaken the request to make a test pass. Report an unsupported capability or ask about a real product decision.

## Completion gate

Before claiming success, verify all of the following:

```bash
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

State what was built, what was tested, whether it was published, and any user-owned connection or product choice still pending.
