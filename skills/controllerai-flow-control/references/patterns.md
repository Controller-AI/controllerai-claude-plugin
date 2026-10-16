<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Flow-control patterns

## Guard a side effect

`only-when` means "skip this action, nothing else":

```text
Read record -> Send notification [only when status changed]
```

Test true and false; the false run shows `only_when_false` on that node and downstream stays eligible.

## Exclusive decision

If/Else when exactly one outcome must occur:

```text
Score -> If/Else
           high priority -> Escalate
           normal        -> Queue
           Else          -> Record unknown
```

Most specific condition first; Else is the fallback and never gets a complement condition.

## Independent fan-out

Router when every applicable action should run:

```text
New order -> Router
              international -> Customs review
              high value    -> Fraud review
              gift          -> Gift note
```

One order may take all three routes. If that is not acceptable, use If/Else.

## Grouped conditions

Outer rows are OR, entries within a row are AND, and each string compiles on its own:

- `node_only_when_set` — `nodeId`, `clear`, `groups`, `js`, `workflowId`
- `branch_update` — `nodeId`, `conditionJson`, `js`, `name`, `branchId`, `workflowId`

## Per-item subflow

- `node_get` — `nodeId`, `workflowId`
- `flow_for_each_scaffold` — `continueOnError`, `flow`, `from`, `itemField`, `itemName`, `itemType`, `listField`, `listJs`, `maxItems`, `name`, `sourceOutput`, `workflowId`

The scaffold creates the item flow, its typed item input, the loop node and the list wiring in one operation, then materializes the optional cap; a failure setting the cap does not roll the scaffold back. The body receives one item, not the whole list; a custom-shaped item and a stored record (`dataRecord.custom.*`) are different types. Continue past a failed iteration (`continueOnError`) only when partial completion is acceptable and the downstream logic handles failed iteration records (null outputs with a status and an error); it cannot rescue a list or mapping expression that fails before the child starts.

## Converge exclusive paths

- `node_add` — `after`, `connection`, `enableProp`, `flow`, `key`, `name`, `node`, `output`, `props`, `workflowId`
- `node_prop_set` — `nodeId`, `js`, `label`, `prop`, `rawExpressionJson`, `value`, `workflowId`, `valueJson`
- `node_prop_list` — `nodeId`, `workflowId`
- `node_prop_set` — `nodeId`, `js`, `label`, `prop`, `rawExpressionJson`, `value`, `workflowId`, `valueJson`
- `node_connect` — `flow`, `from`, `output`, `to`, `workflowId`
- `node_add` — `after`, `connection`, `enableProp`, `flow`, `key`, `name`, `node`, `output`, `props`, `workflowId`

Set `pathDataType` from the returned options once `chooseActivePath` reveals it. Repeat the tail connection for every type-compatible exclusive path; if the shared action already exists, connect it after Join Paths instead of adding a replacement. `chooseActivePath` allows at most one active input and stops the flow when none is active unless `continueIfNoActivePath` is enabled (then it passes null); after a Router whose routes can overlap use `passThrough`, which returns one field per connected input with the blocked ones null. Run one test per branch and stop after all pass.

## Return a flow value

- `flow_output_add` — `flowId`, `description`, `display`, `name`, `type`, `workflowId`
- `node_add` — `after`, `connection`, `enableProp`, `flow`, `key`, `name`, `node`, `output`, `props`, `workflowId`
- `node_prop_list` — `nodeId`, `workflowId`
- `expr_context` — `nodeId`, `methods`, `prop`, `workflowId`
- `node_prop_set` — `nodeId`, `js`, `label`, `prop`, `rawExpressionJson`, `value`, `workflowId`, `valueJson`

Declare the output before adding the Return node so its generated prop exists. Do not guess `registry_flow-return` or `registry_flow-response`. Run the body flow and check the promised top-level output. Populate the outputs on every intended return path, including the "nothing found" path, which should return a clear result without performing writes.

## Error path

Continue-on-error suits steps whose failure must not block the main path: notifications, logging, optional enrichment. The node's error is captured, downstream nodes run with null for its output, and the run ends `completed_with_error`.

- `node_continue_on_error_set` — `nodeId`, `enabled`, `workflowId`
- `node_connect` — `flow`, `from`, `output`, `to`, `workflowId`

Wiring the Error output is optional. Never enable continue-on-error on a critical path (validate, charge, confirm must fail fast) and never to make a parent run green. Read the setting back after changing it.
