---
name: controllerai-triggers
description: Discover, configure, create, sample, enable, test, and debug Controller AI triggers. Use when working with schedules, webhooks, email or app events, captured sample payloads, trigger-to-flow mappings, replay, or any question about why an automation did or did not start.
---

# Controller AI triggers

The `cai ...` examples are canonical grammar. Use the transport selected in `controllerai-start` and its MCP tool map; never mix transports in one task.

A trigger targets exactly one flow. Its source may capture events before publish, but ordinary event execution uses the live workflow. Creating the source is therefore separate from mapping, development testing, publishing, and live replay.

## Require a real source decision

Determine how the trigger choice was established:

- `explicit-in-request`: user named it in the request
- `user-answer`: user chose after clarification
- `inferred-from-existing-workflow`: preserving an existing source

Do not use `assumption`; the backend rejects it. If the request does not determine when the automation runs, ask one focused question before creating a source.

## Discover the exact event

```bash
cai trigger search "<event intent>" --json
cai trigger search --node <nodeSlug> --events-per-node 50 --json
```

Copy the exact `nodeSlug` and `eventKey`. Never infer an event key from its display name.

If authentication is required:

```bash
cai connection list --node <nodeSlug> --json
cai connection connect <nodeSlug>
```

MCP: call `connection_connect_url`; it returns the URL immediately and has no `open` or `wait` parameter.

## Resolve trigger props

```bash
cai trigger props \
  --node <nodeSlug> \
  --event <eventKey> \
  --connection <connectionId> \
  --json
```

Use the returned shape-correct `propsExample` as a starting point. Hidden props remain unset. `readyForCreate` must be true before using it in `trigger create`; the response intentionally omits unresolved required option props rather than emitting a plausible placeholder.

For each name in `unresolvedOptionProps`, run the complete canonical option command returned by the transport, or equivalently:

```bash
cai trigger options \
  --node <nodeSlug> \
  --event <eventKey> \
  --prop <propName> \
  --connection <connectionId> \
  --configured '<configuredPropsJson>' \
  --dynamic-props-id <dynamicPropsId> \
  --query "<human option label>" \
  --json
```

Copy the exact returned `value`, not its display `label`, into `--configured`, then rerun `trigger props`. Use `--page` and returned `context` via `--prev-context` only when search is unavailable.

When a selector has `reloadProps`, rerun with its configured values and returned `dynamicPropsId` before creating the trigger.

```bash
cai trigger props \
  --node <nodeSlug> \
  --event <eventKey> \
  --connection <connectionId> \
  --configured '<configuredPropsJson>' \
  --dynamic-props-id <dynamicPropsId> \
  --json
```

Keep the same node, event, connection, and configured values on every reload, but always pass the LATEST `dynamicPropsId` the backend returned — a reload can mint a new ID, and reusing a stale one resolves against a stale schema. The response provides the complete canonical command; do not shorten it to only `--configured`.

Timer values are objects:

```json
{"cron":{"cron":"0 9 * * *","timezone":"America/Chicago"}}
```

or

```json
{"cron":{"intervalSeconds":900}}
```

Do not discard timezone by passing a bare cron string. The timezone is the user's to
choose, not yours to assume: when they say "at 9am" without one, ask — a schedule in the
wrong timezone runs at the wrong time every single day, and nothing ever errors.

## Create and map

```bash
cai trigger create \
  --flow <targetFlowId> \
  --node <nodeSlug> \
  --event <eventKey> \
  --source-basis <basis> \
  --connection <connectionId> \
  --props '<json>' \
  --dynamic-props-id <id> \
  --json
```

Capture all returned IDs. `triggerId` is the canvas ID used for get/rename/set-enabled/delete. `backendTriggerId` is numeric and used for captured events and replay. They are not interchangeable.

Read a single trigger back with `cai trigger get <triggerId> --json` — the full record: node, event, connection, props, lifecycle. Read-back is always `get`; the CLI has no `describe` read command anywhere (`cai workflow set-description` is a setter).

Follow returned `nextSteps` in order. Trigger payload does not automatically populate the target flow; define flow inputs, relink the run-subflow node if needed, and configure its mapping props.

## Remove or disable an automatic start

`cai trigger delete <triggerId>` removes the automatic start from the development canvas; use it when the start should no longer exist. `cai trigger set-enabled <triggerId> --enable|--disable` changes the enabled state reversibly and idempotently; pass exactly one state flag.

MCP: call `trigger_set_enabled` with `enabled: true` or `enabled: false`.

Both are development edits. Publish after either so production follows; until then, the previous live trigger can still start the published workflow.

## Establish a typed sample

After sending a representative event to the source:

```bash
cai trigger events --trigger <backendTriggerId> --json
cai trigger event <eventId> --json
cai trigger select-event --node <triggerNodeId> --event <eventId> --json
```

Then re-read expression context on downstream nodes. A sample without a platform data type may remain untyped.

## Finish trigger setup: disarm test events

Armed test events let real events reach the DEV version while you set the trigger up. They exist for one purpose: capturing the real data shape so it becomes the trigger node's output type. Arming is the START of trigger setup; **disarming is how trigger setup ENDS.** Once the shape is captured, selected, and typed, turn it off — also when a build fails or is abandoned midway:

```bash
cai trigger test-events --id <triggerId> --disallow
```

MCP: call `trigger_test_events_set` with `allowed: false`, or `allowed: true` to arm it.

The armed state is sticky: nothing disarms it for you, and dev dispatch and live dispatch are independent gates. A trigger that is armed while the workflow is also live runs ONE real event TWICE — both executions real, both billed. The starkest case is a schedule trigger: armed + enabled + published, the cron fires BOTH dev and live at every interval, forever, with no external event involved — until someone edits and republishes.

Keeping a trigger armed alongside live on purpose (side-by-side comparison on real events) is legitimate but uncommon; it is a user decision, never a default.

## Verify development, then publish

Use `cai run flow` for direct development verification, then replay one captured event through the development trigger path. MCP `run_flow` calls take `waitSeconds`.

```bash
cai trigger replay --trigger <backendTriggerId> --event <eventId> --target dev --json
cai exec tree <workflowExecutionId> --json
```

A development replay never dispatches live, but nodes still call real providers and can create side effects or spend usage. Publish only after mappings and development behavior pass.

Before publishing, audit EVERY trigger in the workflow — not just the one you worked on. Users forget triggers left enabled or armed in other flows. Read each row's lifecycle fields from `cai trigger list --json` (`devCanvasEnabled`, `deployment.liveCanvasEnabled`, `deployment.inLiveVersion`, `deployment.devDispatchArmed`, `deployment.dualDispatchLive`) and confirm with the user which triggers should be starting this workflow live:

```bash
cai trigger list --json
cai workflow publish --name "<release>" --json
```

If publish fails because a trigger still has test events armed, that error is the designed backstop, not a malfunction: trigger setup was never finished. The normal resolution is to disarm and republish. Publishing WITH an armed trigger requires the user's explicit, informed choice — explain the double-run and double-billing consequence first, then pass the intent flag:

```bash
cai workflow publish --name "<release>" --acknowledge-dual-dispatch --json
```

Never pass `--acknowledge-dual-dispatch` on your own initiative.

Live replay is optional and requires an explicit acknowledgement:

```bash
cai trigger replay \
  --trigger <backendTriggerId> \
  --event <eventId> \
  --target live \
  --confirm-live \
  --json
cai exec tree <workflowExecutionId> --json
```

It executes the published workflow, calls live third-party actions, and spends usage. Obtain appropriate user intent before replaying a side-effectful event.

Read [references/lifecycle.md](references/lifecycle.md) for trigger state and debugging.
