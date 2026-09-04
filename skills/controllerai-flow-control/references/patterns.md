# Flow-control patterns

## Guard a side effect

Use `only-when` when false should mean "skip this action" and nothing else:

```text
Read record -> Send notification [only when status changed]
```

Test both true and false. A false run should show an intentional skip reason.

## Exclusive decision

Use If/Else when only one outcome must occur:

```text
Score -> If/Else
           high priority -> Escalate
           normal        -> Queue
           Else          -> Record unknown
```

Put the most specific/highest-priority conditions first. Never write a complement condition for Else.

## Independent fan-out

Use Router when every applicable action should run:

```text
New order -> Router
              international -> Customs review
              high value    -> Fraud review
              gift          -> Gift note
```

One order may take all three routes. If that is not acceptable, use If/Else.

## Grouped branch condition

Only Router and If/Else branch commands accept grouped conditions. The outer rows are OR and the expressions within one row are AND:

```bash
cai branch update <controlNodeId> \
  --route <routeId> \
  --condition-json '[["flow.email.isNotEmpty()","flow.units < 5"],["flow.override === true"]]' \
  --json
```

Each string must compile independently. Node `only-when` has no `--condition-json`; if exact compound node-only semantics are required, report the limitation instead of using a private API or weakening the guard.

## Per-item subflow

Use `cai flow for-each` to create the item flow, typed item input, loop node, and list wiring atomically. First run `cai node get <sourceNodeId> --json`, choose the intended `outputs[]` entry, and pass its exact opaque `id` as `--source-output`; never pass its label or assume the first output. Omit list-selection flags when the whole selected output payload is a list; use `--list-field <nested.path>` for a list below `.data`; reserve `--list-js` for an exact advanced expression or `--js-file <path|->` for that expression in a UTF-8 file/stdin. The three selection forms are mutually exclusive.

MCP: call `flow_for_each_scaffold` and pass expression source inline as `listJs`.

The runtime limit defaults to 500 items and supports 1 through 10000. Use `--max-items` when the intended bound differs, then verify the returned read-back. Put meaningful work inside the generated body flow and declare body outputs before consuming aggregate results.

Decide explicitly whether one child-flow failure should stop the sequential loop. Use `--continue-on-error` only when partial completion is acceptable and downstream logic handles failed iteration records. It does not recover a list or mapping expression that fails before an iteration starts.

## Converge exclusive paths

Initialize Join Paths and select its active-path mode:

```bash
cai node init \
  --flow <flowId> \
  --node registry_flow \
  --key registry_flow-merge \
  --name "Join paths" \
  --json
cai node set <joinNodeId> --prop mode --value chooseActivePath --json
cai node props <joinNodeId> --json
# If returned props require it:
cai node set <joinNodeId> --prop pathDataType --value <returnedTypeValue> --json
cai node connect --flow <flowId> --from <firstBranchTailNodeId> --to <joinNodeId> --json
cai node connect --flow <flowId> --from <secondBranchTailNodeId> --to <joinNodeId> --json
cai node init \
  --flow <flowId> \
  --node <sharedNodeSlug> \
  --key <sharedNodeKey> \
  --name "<shared action>" \
  --after <joinNodeId> \
  --json
```

Set `pathDataType` only when returned props require it, using an exact returned option. Repeat the branch-tail connection for every type-compatible exclusive path. If the shared action already exists, connect it after Join Paths instead of initializing a replacement. Do not connect several branches directly to an ordinary node or duplicate the action. `chooseActivePath` requires at most one active input, so do not use it after a Router whose routes can overlap. Run one test per branch and stop after all pass.

## Return a flow value

Declare the output before the Return response node so its generated prop exists:

```bash
cai flow output add <bodyFlowId> \
  --display "<output label>" \
  --name <output_key> \
  --type <type> \
  --json
cai node init \
  --flow <bodyFlowId> \
  --node registry_flow \
  --key registry_flow-return-response \
  --name "Return <value>" \
  --after <producerNodeId> \
  --json
cai node props <returnNodeId> --json
cai expr context <returnNodeId> --prop <generatedOutputProp> --json
cai expr set <returnNodeId> --prop <generatedOutputProp> --js '<producer binding>' --json
```

Do not guess `registry_flow-return` or `registry_flow-response`. Run the body flow and verify its top-level output contains the promised value.

## Error path

Continue-on-error is for steps whose failure must not block the main path — notifications, logging, optional enrichment. Enabling it captures the node's error, downstream nodes still execute (receiving null for the failed node's output), and the run ends `completed_with_error`:

```bash
cai node continue-on-error <nodeId> --enable --json
```

MCP: call `node_continue_on_error_set` with `enabled: true` or `enabled: false`.

Wiring the Error output is OPTIONAL — add it when a failure needs explicit handling, not as a precondition:

```bash
cai node connect --flow <flowId> --from <nodeId> --to <handlerNodeId> --output error --json
```

Never enable continue-on-error on critical-path steps (validate → charge → confirm must fail fast), and never merely to make the parent execution green. Read back the setting after changing it.
