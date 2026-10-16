---
name: controllerai-triggers
description: Discover, configure, create, capture, map, test, enable and debug Controller AI triggers: schedules, webhooks, email and app events. Use when an automation must start without a chat message, when a captured payload must become typed flow input, when replaying an event, or when asking why an automation did or did not start.
---
<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->

# Triggers

## Use this when

- A workflow must start on a schedule, a webhook, an email or an app event.
- A captured event must become the typed input of a flow, or a trigger's mapping is wrong.
- A trigger must be paused, resumed, retired or replaced, or a run did not start.

Flow inputs and mapping expressions belong to `controllerai-expressions`; reading a run belongs to `controllerai-runs`; publishing belongs to `controllerai-workflows`.

## How it works

**One trigger, one flow.** A trigger binds a source (node slug plus exact event key, its connection and props) to exactly one flow, and its payload does not populate that flow by itself: the flow needs declared inputs, and the mapping props live on the trigger canvas's run-subflow node (`runSubflowNodeId`, wired automatically, never connected by hand), one per flow input, synthesized after the inputs exist. Creating the source, capturing and selecting a sample, mapping, testing on development, publishing and verifying live are separate steps.

**Two rails, three switches.** Events are captured from the moment the source exists, and ordinary event execution runs the live release, so a trigger does not run live until the workflow is published; before that it can execute on development only through a replay or armed ambient dispatch. `set-enabled` (enable or disable) is the trigger's definition state and edits development only; the prior live trigger keeps running until an authorized publish. `test-events` (allow or disallow) sets a sticky permission bit for ambient development dispatch: real events then also execute the development canvas. `replay` targets one rail explicitly. The bit survives a disable (`devDispatchArmed` is enabled-and-allowed), and once the workflow is published with the trigger enabled on both rails and routable, every real event can execute on both rails (a schedule every interval), which is why publish refuses an armed trigger unless the user explicitly acknowledges dual dispatch.

**Four IDs.** The canvas `triggerId` addresses get, rename, set-enabled, delete and test-events. The numeric `backendTriggerId` addresses captured events and replay. The `triggerNodeId` selects the typed sample. The captured `eventId` addresses event, select-event and replay. Copy them from the responses; they are never interchangeable.

**Props are resolved, not guessed.** The props read returns a shape-correct `propsExample`, `readyForCreate`, and `unresolvedOptionProps`; unresolved required option props are omitted rather than filled with a plausible placeholder, and hidden props stay unset (they are provider reload plumbing). Option values are resolved by the options read with the same node, event, connection and configured values. The dynamic schema is identified by the response's `id`, passed back as `dynamicPropsId`; a selector that reloads props can mint a new one, so always pass the latest and keep every earlier configured value on each reload (registry events have no dynamic schema). Timer values are objects, `{"cron":{"cron":"0 9 * * *","timezone":"America/Chicago"}}` or `{"cron":{"intervalSeconds":900}}`, never a bare cron string.

**A sample makes the type.** A captured event becomes the trigger node's output type only when it is selected; the mapping props are bound after that, from the re-read expression context, and a payload without a platform data type can stay untyped.

## Rules

1. **The start condition is the user's decision.** Create a source only with an explicit `sourceBasis`: `explicit-in-request`, `user-answer`, or `inferred-from-existing-workflow` when preserving an existing source. The backend rejects an assumption; when the request does not say when the automation runs, ask one focused question first.
2. **Copy the exact `nodeSlug` and `eventKey`** from the search result; never derive an event key from its display name, because keys are opaque and display names collide across providers.
3. **Ask for the timezone** when the user says "at 9am" without one, and never infer a production cadence from a test sample, whose timing is a test artifact.
4. **Repair in place.** Keep the trigger's canvas ID and source when fixing its mapping or readiness; replacing a trigger changes captured IDs and may change a webhook URL, so replace only when the source itself must change, then recapture every ID.
5. **Delete a retired start; disable a paused one.** "Stop starting itself", "retire" and "no longer" mean delete: a disabled trigger still exists and can be re-enabled, so it is not retired. Never delete the owning flow to remove a trigger (that destroys the graph and its callers), and pass exactly one `enabled` value.
6. **Arm ambient dispatch only on the user's decision, and disarm when setup ends**, including a failed or abandoned setup. Capture needs no arming; arming executes a half-built development flow on real traffic, so explain that before adding the acknowledgement. Keeping both rails armed is an uncommon, explicit user choice, never a default, and routine tests use replay.
7. **Never mix the state parameters or the acknowledgements.** `trigger_set_enabled.enabled`, `trigger_test_events_set.allowed`, `target: "dev" | "live"`; `acknowledgeDualRun` only while arming, `workflow_publish.acknowledgeDualDispatch` only while publishing an armed trigger, `confirmLive` only for one live replay. Never add an acknowledgement on your own initiative.
8. **Audit every trigger before a publish**, not only the one you touched; users forget starts left enabled or armed in other flows.
9. **A captured event is not a run.** For create or mapping work, verify received inputs, outputs and effects on a development replay; an event with no execution and an execution with empty inputs are different faults. Read-only diagnosis, a deletion (verified through the list and status) and a user-declined test need no replay.
10. **A draft-only request stops at development proof**, and a delete-only or disable-only task needs no replay.

## Recipes

### Discover and resolve

- `trigger_search` — `query`, `eventsPerNode`, `limit`, `node`, `workflowId`
- `trigger_component_props` — `configured`, `connection`, `dynamicPropsId`, `event`, `node`, `workflowId`
- `trigger_prop_options` — `configured`, `connection`, `dynamicPropsId`, `event`, `node`, `page`, `prevContext`, `prop`, `query`, `workflowId`
- `trigger_component_props` — `configured`, `connection`, `dynamicPropsId`, `event`, `node`, `workflowId`

Omit the connection for a source without auth. Copy the returned option `value`, not its `label`, into the configured props, then re-read the props until `readyForCreate` is true. `trigger_component_props` takes `node`, `event`, `connection`, `configured` (the object of values chosen so far) and `dynamicPropsId`; `trigger_prop_options` adds `prop`, `query`, or `page` with `prevContext` when a search does not find the option. A missing connection is handled as in `controllerai-node-config`: `connection_list`, then `connection_connect_url` shown as a link.

### Create and map

- `trigger_create` — `connection`, `dynamicPropsId`, `event`, `flow`, `node`, `props`, `sourceBasis`, `workflowId`
- `trigger_get` — `triggerId`, `workflowId`

`trigger_create` takes `flow`, `node`, `event`, `sourceBasis`, `connection`, `props` and `dynamicPropsId`. Capture every returned ID (`triggerId`, `backendTriggerId`, `triggerNodeId`, `runSubflowNodeId`) plus the `webhookUrl` or `emailAddress` when the source has one, and follow `nextSteps` in order: declare the flow inputs, capture and select a sample, then bind one mapping prop per input on the run-subflow node from its re-read expression context. Read-back is `get`; there is no describe command.

### Capture and select a sample

- `trigger_event_list` — `limit`, `triggerId`, `workflowId`
- `trigger_event_get` — `eventId`, `workflowId`
- `trigger_event_select` — `eventId`, `triggerNodeId`, `workflowId`

A representative event has to reach the source: through the provider, the returned webhook URL or email address; never invent an endpoint. Inspect the payload before selecting it and re-read the run-subflow node's expression context after selecting. `trigger_event_list` and `trigger_replay` take the numeric `backendTriggerId` as `triggerId` (a numeric string), unlike every other trigger tool.

### Arm ambient development dispatch (only on request)

- `trigger_test_events_set` — `triggerId`, `acknowledgeDualRun`, `allowed`, `workflowId`

Arming makes every real event also execute the development canvas while it is half built; use it only when the user wants that after hearing it, and disarm at the end of setup. `trigger_test_events_set` takes `allowed: true|false` and `acknowledgeDualRun: true` when arming.

### Pause, resume, retire

- `trigger_set_enabled` — `triggerId`, `enabled`, `workflowId`
- `trigger_delete` — `triggerId`, `workflowId`

`trigger_set_enabled` takes the canvas `triggerId` and `enabled: true|false`; `trigger_delete` has no confirmation parameter.

### Test, audit, publish

- `run_flow` — `flowId`, `confirmLive`, `inputs`, `target`, `waitSeconds`, `workflowId`
- `trigger_replay` — `confirmLive`, `eventId`, `payload`, `target`, `triggerId`, `workflowId`
- `execution_tree` — `workflowExecutionId`, `cursor`, `flowId`, `limit`, `status`, `workflowId`
- `trigger_audit` — `workflowId`
- `trigger_list` — `workflowId`

Run the target flow directly first, then replay one captured event through the development trigger path; a development replay never dispatches live but still calls real providers and spends usage. `trigger_replay` takes `target: "dev"` with `eventId` or a raw `payload`. The audit reports coverage, armed development dispatch and double-run risk; the list and get reads show `devCanvasEnabled` at the top level and `deployment.liveCanvasEnabled`, `deployment.inLiveVersion`, `deployment.devDispatchArmed` and `deployment.dualDispatchLive` (live fields are null when no live canvas exists yet: never published, or this trigger added after the last publish). Publish through `controllerai-workflows` only for an authorized live change. A publish refused because a trigger is still armed is the designed backstop: disarm and republish; publishing armed needs the user's informed choice after hearing about double runs and double billing, and only then `workflow_publish` with `acknowledgeDualDispatch: true`.

### Verify live

- `trigger_replay` — `confirmLive`, `eventId`, `payload`, `target`, `triggerId`, `workflowId`
- `execution_tree` — `workflowExecutionId`, `cursor`, `flowId`, `limit`, `status`, `workflowId`

`trigger_replay` takes `triggerId` (the numeric ID), `eventId`, `target: "live"` and `confirmLive: true`.
A live replay executes the published workflow, calls live third-party actions and spends usage; it needs its own authorization and is optional.

## Verify

- `trigger_get` — `triggerId`, `workflowId`
- `trigger_list` — `workflowId`
- `workflow_status` — `workflowId`

The trigger, source, target flow, connection, props and mapping match the request; `devCanvasEnabled` and `deployment.liveCanvasEnabled` / `deployment.inLiveVersion` match the requested rail state; `deployment.devDispatchArmed` is false unless the user chose dual dispatch; for create or mapping work, the development replay's execution tree shows the flow receiving the mapped inputs and producing the expected effect; for a deletion, the list no longer shows the trigger and the status shows what remains live. Read [references/lifecycle.md](references/lifecycle.md) for the state sequence, the ID table and the missing-run diagnosis.

## Report

The trigger and its source, the IDs the user needs, whether it is enabled on development and on live, whether ambient dispatch is armed and why, what the development replay proved, and what remains: an unpublished change, a live verification not yet run, or a source decision still open.
