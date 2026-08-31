---
name: controllerai-runs
description: Run and debug Controller AI nodes on development and flows on development or the published live version — execution history, ports, outputs, skip reasons, and resolved configs. Use when verifying or testing a build, investigating a failed or green-but-wrong run, resolving an unknown output type, checking expression values, or proving what data and configuration reached each node.
---

# Controller AI runs

In MCP hosts, send each exact literal `cai ...` command through the single `exec` tool; never translate it into another grammar.

Think "real run", not "simulation". `run node` always uses development; `run flow` defaults to development and can target the published live version. Both rails execute real integration actions (messages send, records change, provider usage is spent); development isolation covers the data store only. Running is a required part of building — one successful run is how a dynamic node's output type gets resolved for the next step — so mitigate risk before running.

## Consequential actions: mitigate with the user, don't refuse

Before running an action with external consequences (send, reject, delete, charge), tell the user plainly what will really happen and ask for a safe target:

- Email/message sends → "What address/channel can I send the dev run to?" (often the user's own).
- Record deletes/updates → "Is there a record I can safely use, or shall I create one first and delete that?"
- People-affecting actions (e.g. reject a candidate) → "Which one is safe to use for this run?"

If the user declines running the action, accept it. Then keep the build honest: declare the node's output contract from the provider's documented response —

```bash
cai node declare-output <nodeId> --sample-file shape.json
```

— where `shape.json` is a representative response object from the provider's API reference. The contract is marked `declared (unverified)`; the first real completed run verifies and upgrades it automatically. State clearly that it is unverified until then, and make the first live run the explicit verification step. Never silently skip the question and never present an unrun action as verified.

## First live run is part of development

After publishing, run the published version directly under supervision:

```bash
cai run flow <flowId> --inputs values.json --target live --confirm-live --json
```

This command fires external actions and spends usage. For risky sends, build the safe-audience harness: branch on the `isLiveVersion` data source with If/Else — the false branch targets the developer (own email/channel), the true branch targets the real audience. Use this direct live run for explicit verification; do not wait for a real trigger event.

## Prepare exact run inputs

```bash
cai run inputs <nodeId> --json
cai run inputs <flowId> --flow --json
```

Use returned opaque input IDs. Flow mode lists EVERY declared input with a `used` flag: supply the used ones (an unsupplied used input falls back to its saved test value), while values keyed to unused inputs are accepted and ignored. A compact input file can be a flat map:

```json
{
  "<inputId>": "representative value"
}
```

Do not key test values by a guessed display name.

Before an isolated node run, `cai run inputs <nodeId> --json` calls `get_node_test_inputs`; supply every referenced `flowInputs[].id` explicitly. Missing one fails before execution with `ISOLATED_FLOW_CONTEXT_REQUIRED`. Explicit `null`, `""`, `false`, and `0` are valid supplied values.

Date-typed inputs take plain ISO-8601 strings, e.g. `"2026-06-05"` or `"2026-06-05T12:30:00Z"` (no double-encoding, no epoch strings).

## Run the smallest useful scope

Start with a changed node, then run its containing flow:

```bash
cai run node <nodeId> --inputs values.json --json
cai run flow <flowId> --inputs values.json --json
```

`run node` is deliberately isolated: it does not execute wired upstream nodes. Read its `runScope`. If `endToEndValidationRequired` is true, values derived from upstream can be empty even when the selected node is green; do not accept that run as expression proof. `run flow` is the end-to-end check that executes upstream wiring.

The CLI waits for a terminal execution state. Capture `workflowExecutionId` and `nodeExecutionId` from the result.

For every node test, inspect:

- `status` and `errorMessage`
- `skipReason`
- `runScope`, especially `upstreamNodesExecuted` and `endToEndValidationRequired`
- `resolvedConfigs`
- `resolvedConfigs[].resolutionWarnings`: trust every structured entry; `[]` is authoritatively clean. Runs also surface entries on stderr, so read stderr even when the execution completed.
- `outputTypeAdoption`: the server verifies the exact completed `nodeExecutionId`, derives its persisted output shape, and updates only that non-static node. `adopted` or `already_typed` lets the next `node status` converge; `no_observed_output` means the representative run returned no reusable shape. `adoptedNodeIds` must be empty or contain only the tested node; stop if it names a sibling.
- output shape and values

A dynamic prop with a null/empty resolved config is a failure even when status is green. `0` and `false` are valid resolved values.

## Debug from the execution tree

```bash
cai exec tree <workflowExecutionId> --json
cai exec get <workflowExecutionId> --json
cai exec flow <flowExecutionId> --json
cai exec node <nodeExecutionId> --json
cai exec ports <nodeExecutionId> --json
```

Use the tree to locate the first unexpected status. Use ports to compare actual inputs, resolved configuration, and outputs at that node. Read [references/debugging.md](references/debugging.md) for the fault-isolation order.

## Output discovery

Prefer a configured node test. If a component is not yet in the graph, `run discover` creates a scratch node and deletes it after execution:

```bash
cai run discover \
  --node <nodeSlug> \
  --key <nodeKey> \
  --connection <connectionId|none> \
  --props '<json>' \
  --json
```

This calls the real third-party action and spends usage. For a destructive action, do not probe merely to learn its output — first ask the user for a safe target (see "Consequential actions" above); if they decline, declare the expected shape instead. Use `--keep` only when the user wants the scratch node retained.

After successful output discovery, re-read `cai node status` and downstream `cai expr context`; verify the adopted output type is concrete.

## Completion gate

- Gate scripts on `cai workflow validate`; it exits 1 when the response has `valid: false`.
- The test exercised the changed path with credible values.
- The first unexpected node, if any, was identified rather than inferred from the final error.
- All dynamic configs resolved to intended values.
- Output fields and types match downstream expectations.
- Branch skips are intentional and error paths are accounted for.
- A full flow test passes after isolated node fixes.
- `completed` with an empty or null output is treated as a failure until the output values are read and explained — an empty result usually means a mis-mapped input or an expression that null-propagated silently.
- The final report cites execution IDs and distinguishes development testing from live verification.
