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

```bash
cai node only-when <nodeId> --groups-json '[["flow.email.isNotEmpty()","flow.units < 5"],["flow.override === true"]]' --json
cai branch update <controlNodeId> --route <routeId> --condition-json '[["flow.email.isNotEmpty()","flow.units < 5"],["flow.override === true"]]' --json
```

## Per-item subflow

```bash
cai node get <sourceNodeId> --json
cai flow for-each --flow <parentFlowId> --from <sourceNodeId> --source-output <exactOutputId> --name "<item flow name>" --item-type <type> --json
```

The scaffold creates the item flow, its typed item input, the loop node and the list wiring in one operation, then materializes the optional cap; a failure setting the cap does not roll the scaffold back. The body receives one item, not the whole list; a custom-shaped item and a stored record (`dataRecord.custom.*`) are different types. Continue past a failed iteration (`--continue-on-error`) only when partial completion is acceptable and the downstream logic handles failed iteration records (null outputs with a status and an error); it cannot rescue a list or mapping expression that fails before the child starts.

## Converge exclusive paths

```bash
cai node add --flow <flowId> --node registry_flow --key registry_flow-merge --name "Join paths" --json
cai node set <joinNodeId> --prop mode --value chooseActivePath --json
cai node props <joinNodeId> --json
cai node set <joinNodeId> --prop pathDataType --value <returnedTypeValue> --json
cai node connect --flow <flowId> --from <firstBranchTailNodeId> --to <joinNodeId> --json
cai node connect --flow <flowId> --from <secondBranchTailNodeId> --to <joinNodeId> --json
cai node add --flow <flowId> --node <sharedNodeSlug> --key <sharedNodeKey> --name "<shared action>" --after <joinNodeId> --json
```

Set `pathDataType` from the returned options once `chooseActivePath` reveals it. Repeat the tail connection for every type-compatible exclusive path; if the shared action already exists, connect it after Join Paths instead of adding a replacement. `chooseActivePath` allows at most one active input and stops the flow when none is active unless `continueIfNoActivePath` is enabled (then it passes null); after a Router whose routes can overlap use `passThrough`, which returns one field per connected input with the blocked ones null. Run one test per branch and stop after all pass.

## Return a flow value

```bash
cai flow output add <bodyFlowId> --display "<output label>" --name <output_key> --type <type> --json
cai node add --flow <bodyFlowId> --node registry_flow --key registry_flow-return-response --name "Return <value>" --after <producerNodeId> --json
cai node props <returnNodeId> --json
cai expr context <returnNodeId> --prop <generatedOutputProp> --json
cai expr set <returnNodeId> --prop <generatedOutputProp> --js '<producer binding>' --json
```

Declare the output before adding the Return node so its generated prop exists. Do not guess `registry_flow-return` or `registry_flow-response`. Run the body flow and check the promised top-level output. Populate the outputs on every intended return path, including the "nothing found" path, which should return a clear result without performing writes.

## Error path

Continue-on-error suits steps whose failure must not block the main path: notifications, logging, optional enrichment. The node's error is captured, downstream nodes run with null for its output, and the run ends `completed_with_error`.

```bash
cai node continue-on-error <nodeId> --enable --json
cai node connect --flow <flowId> --from <nodeId> --to <handlerNodeId> --output error --json
```

Wiring the Error output is optional. Never enable continue-on-error on a critical path (validate, charge, confirm must fail fast) and never to make a parent run green. Read the setting back after changing it.
