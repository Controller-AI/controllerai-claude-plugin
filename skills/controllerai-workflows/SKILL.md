---
name: controllerai-workflows
description: Build, inspect, edit, recover, and publish Controller AI workflows and their flows. Use when a user asks to create or change an automation, a flow, or a workflow release, and for the lifecycle rules every other workflow skill relies on.
---
<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->

# Workflows

## Use this when

- Creating a workflow, adding or repairing flows, orienting inside an existing graph, validating, checkpointing, restoring, publishing, or deleting.
- Another skill needs the shared lifecycle rules: which rail an edit lands on, how contracts and IDs behave, what validation and publishing require.

Load the narrow skill when the work reaches it: nodes, connections, props and wiring → `controllerai-node-config`; computed values and conditions → `controllerai-expressions`; Router, If/Else, loops, joins, subflows → `controllerai-flow-control`; runs and debugging → `controllerai-runs`; schedules, webhooks, app events → `controllerai-triggers`; schemas and records → `controllerai-data`; Ask AI and agent nodes → `controllerai-ai-nodes`; user-facing agents → `controllerai-agents`.

## How it works

A **workflow** groups flows, custom schemas, data collections and triggers, and its flows share one data store per environment. A **flow** is one canvas and one callable contract: typed inputs, connected nodes, typed outputs. A Return node maps values into the declared outputs; declaring an output does not fill it. Workflow, flow, node, input and output IDs are distinct identities; never swap one for another.

Every graph edit lands on the **development** version. Publishing creates the **live** release; a live agent that uses the workflow as a tool follows its latest live release without an agent publish, while the owner's Test conversations run the development version. Publishing never copies development records into the live store, and every environment keeps its own data.

Every save is a **state**. History lists states in groups and gives candidate state IDs; a restore rolls the definition back to one state and leaves records and external effects untouched. How far back history is listed and how far back a state can be restored are two separate plan limits, so a listed state is not always restorable.

Deleting a workflow tears down its deployed triggers and soft-deletes it. It is refused while any draft or live agent depends on the workflow, and nothing undeletes. The dependency map shows a workflow's agent and workflow callers and the connections it uses. Deleting a flow removes its graph, its contract and the triggers targeting it, and breaks every caller bound to it.

The account event feed records only supported events: connections created, workflows created, updated, deleted, published and run, agents created and published, members added. Settings changes, member removals and label edits are not in it, and a past change that is not recorded cannot be reconstructed from names or current state.

## Rules

1. **Before changing an existing workflow, read its outline and the affected flow**, then repair those IDs rather than creating a replacement workflow or flow. A repair that recreates loses history, triggers and callers.
2. **Rename contracts in place.** Deleting and recreating an input or output breaks every expression, subflow mapping and agent tool bound to it. After a rename or deletion, re-read the flow and repair the affected dependents before validating or running; a stored input-ID binding survives a rename, so the readback may show nothing to repair.
3. **"Store, save, log, record" without a named destination means a custom data type plus a create-record node.** A flow output alone persists nothing. When the user names a destination (a sheet, a CRM, an app, a file), write there through that integration; add a platform record only if they also want a copy kept here.
4. **Validate after structural edits and before publishing.** Fix every listed prop, wire, expression, type, subflow, branch and trigger error, or report each one with what it breaks. Publish refuses errors; `acknowledgeValidationErrors` is for issues the user has seen and accepted, never a shortcut. If the user forbids a repair, report and stop.
5. **Validation counts are status.** Do not write them into unrelated workflow properties unless the user asks for exactly that.
6. **Deleting a workflow needs the user's explicit intent.** A confirmation flag is not authorization. Check dependencies first; detach draft tools with `agent_tool_remove`, and treat a live agent's republish as its own decision.
7. **Layout once per changed flow.** After adds, deletes or rewires, run the flow layout as the last structural step; placement at add time only avoids overlap.
8. **Never weaken the request to make a test pass.** Report an unsupported capability or ask the product question. Keep deliberately armed triggers and advisory-only nodes the user chose.
9. **A conflict means re-read, then reapply the intent to the current graph.** Never replay a stale document or old IDs; the retried write uses the IDs the current state holds.
10. **Say "not recorded" and change nothing** when asked to reconstruct history the feed does not hold.

## Recipes

### Establish context

Call `account_status`, select the workflow from `workflow_list`, then call `workflow_status` with its exact `workflowId`. There is no pin: capture the ID once and pass it on every workflow-scoped call.

### Create

- `workflow_create` — `name`

Create with the final name. Read `flowCount` and `nextStep`: with 0 flows, create the first flow with `flow_create`; with one empty default flow, rename it and build in it; when either field is missing, list the flows before deciding. Add flows only for genuinely separate canvases, and stop at an empty workflow when that is what was asked. A missing projected field is not a failed create: read the full result or the exact inventory before retrying.

### Orient

- `workflow_outline` — `workflowId`
- `workflow_state_get` — `flowId`, `stateId`, `workflowId`
- `flow_get` — `flowId`, `workflowId`

The outline shows flows, contracts, graph dependencies, triggers and data inventory in one call. State is a bounded projection; pass the flow ID for node and edge identities. The flow read exposes props and expressions.

### Build

1. Translate the request into inputs, steps, outputs, trigger, integrations, failure behavior and publish intent.
2. Declare flow inputs and outputs before nodes whose props derive from them. Pass a human display label; use the current binding from `flow get` (`jsBinding`) or the outline (`binding`), never the diagnostic `legacyBinding`.
3. Add nodes in execution order and name each after the data it produces; downstream bindings are seeded from that name when wired.
4. Lay out each changed flow, validate, test on development, re-read status. Publish only when the user asked for a live change or authorized deployment.

### Publish

- `workflow_status` — `workflowId`
- `trigger_list` — `workflowId`
- `workflow_publish` — `acknowledgeDualDispatch`, `acknowledgeValidationErrors`, `name`, `workflowId`

Read status and every trigger first: publishing activates enabled triggers on the live release. The publish refuses validation errors and may return an approval that the user must grant.

`workflow_publish` takes `workflowId`, `name`, `acknowledgeValidationErrors` and `acknowledgeDualDispatch`. When it returns an approval, continue it with `approval_resume` (`approvalId`, `waitSeconds`) rather than publishing again.

### Checkpoint and recover

- `workflow_state_save` — `name`, `workflowId`
- `workflow_history` — `group`, `version`, `workflowId`
- `workflow_state_restore` — `stateId`, `version`, `workflowId`

Save before a risky rewrite. Expand a group to find the state ID; restore by state, not by group.

### Delete

- `dependency_map` — `workflowId`
- `workflow_delete` — `workflowId`
`dependency_map` takes an optional `workflowId`; omit it for the account-wide map. `workflow_delete` has no confirmation parameter, which changes nothing about needing the user's intent.

## Verify

- `workflow_outline` — `workflowId`
- `workflow_validate` — `workflowId`
- `workflow_status` — `workflowId`
- `flow_get` — `flowId`, `workflowId`
- `run_flow` — `flowId`, `confirmLive`, `inputs`, `target`, `waitSeconds`, `workflowId`

Pass the parsed values as `inputs` and set `waitSeconds` on `run_flow`.

Evidence that a build works: validation reports zero errors, or each remaining error is reported with its effect; required connections and props are complete; expressions resolve to the expected values, with intentional empties explained; the expected branch and error paths ran where practical; the flow outputs contain the promised values; the development or live state matches the request; the final outline matches the request, with only your own abandoned scratch objects removed. A checkpoint or a rename needs its readback, not a run. Once the requested blockers are fixed and each changed path ran once, stop; warnings alone do not justify a redesign.

## Report

What was built, what ran and what it returned, whether it is published, and any connection or product decision still pending. Share returned product links only; never guess one.

