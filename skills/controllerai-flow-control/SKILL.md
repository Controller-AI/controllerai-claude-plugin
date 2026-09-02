---
name: controllerai-flow-control
description: Design and configure Controller AI execution paths using only-when rules, Router routes, If/Else branches, for-each loops, and subflows. Use when work must branch, fan out, skip, iterate over a list, call another flow, converge paths, or handle errors without changing the user's intended semantics.
---

# Controller AI flow control

The `cai ...` examples are canonical grammar. Use the transport selected in `controllerai-start` and its MCP tool map; never mix transports in one task.

Choose the primitive from the intended execution semantics before adding nodes.

## Pick the correct primitive

| Intent | Primitive | Semantics |
| --- | --- | --- |
| Skip one action when a condition is false | Node `only-when` | No alternate path |
| Choose exactly one path | If/Else | First matching branch runs; Else is last |
| Run every matching path | Router | Conditions are independent; multiple routes may run |
| Apply work to every list item | `flow for-each` | Atomic loop scaffold with typed item flow |
| Reuse a typed unit of work once | Run flow/subflow node | Explicit input and output contract |
| Continue after an action error | Error output or continue-on-error | Error behavior is explicit and testable |

Do not substitute one primitive merely because its component is easier to find.

## Conditions

Before writing any condition:

```bash
cai expr context <controlNodeId> --json
cai expr validate <controlNodeId> --mode onlyWhen --js '<boolean expression>' --json
```

For one guarded action:

```bash
cai node only-when <nodeId> --js '<boolean expression>' --json
```

Use `--clear` only when the user's intent is to remove the guard.

Choose the primitive by the runtime signal that callers and tests need. A false node `only-when` records `skipReason: "only_when_false"`; a node on an untaken If/Else branch or unmatched Router route records `skipReason: "upstream_skipped"`.

## If/Else and Router

Discover and initialize the appropriate control component using `controllerai-node-config`, then inspect its exact branch IDs:

```bash
cai branch list <controlNodeId> --json
```

A fresh If/Else already contains one blank `If` branch and an Else branch. The blank condition is a publish/runtime blocker: copy that default branch ID from `branch list` and update it before adding another condition branch. Do not leave the default blank and create a replacement beside it.

```bash
cai branch update <ifElseNodeId> \
  --route <defaultIfBranchId> \
  --name "<meaning>" \
  --js '<boolean expression>' \
  --json
```

Add another condition only when the decision needs another non-Else case. Router routes are independent, so update an existing blank route or add each intended route explicitly:

```bash
cai branch add <controlNodeId> --name "<meaning>" --js '<boolean expression>' --json
cai branch update <controlNodeId> --route <routeOrBranchId> --js '<boolean expression>' --json
cai branch connect <controlNodeId> --route <routeOrBranchId> --to <targetNodeId> --flow <flowId> --json
```

For If/Else, order is behavior:

```bash
cai branch reorder <ifElseNodeId> --order <conditionId1,conditionId2> --json
```

List every condition branch exactly once and omit Else; the operation keeps Else last. Router routes cannot be reordered because every match runs.

After edits, run `cai branch list` again. Treat its warnings about missing conditions or unwired paths as blockers: an unwired selected path can end with an apparently green execution.

## Loops

Never assemble the standard per-item subflow pattern by hand:

```bash
cai node get <listProducingNodeId> --json
cai flow for-each \
  --flow <parentFlowId> \
  --from <listProducingNodeId> \
  --source-output <exactOutputId> \
  --name "<item flow name>" \
  --item-type <text|number|date|boolean|custom.type> \
  --json
```

Use repeatable `--item-field <id:type[:display]>` only when creating a new `custom.*` item type.

Copy `--source-output` from the selected node's exact `outputs[].id`. It is an opaque ID, not `displayName`, and there is no first-output default. This matters for Router, If/Else, and error outputs as well as ordinary action nodes.

- If the source's whole output is the list, omit both `--list-field` and `--list-js`.
- If the list is nested in the source payload, use `--list-field` with the dot-delimited path below `.data`; the operation derives the not-yet-created loop input binding deterministically.
- Use `--list-js` only for an advanced exact expression. For multiline or shell-sensitive source, use `--js-file <path|->` instead. Both are mutually exclusive with `--list-field` and with each other; copy the wrapper/binding shape from expression context rather than guessing it.

MCP: pass expression source inline as `listJs` to `flow_for_each_scaffold`; the server cannot read a local `--js-file` path.

- The runtime cap defaults to 500 items and accepts 1 through 10000. Set `--max-items` explicitly when the product requirement differs. The scaffold operation creates the core graph first, then materializes and verifies this optional prop. The cap is all-or-nothing at runtime: a list larger than the cap throws and NOTHING iterates — there is no partial truncation.

Changing an existing loop: update its loop-node props directly:

```bash
cai node set <loopNodeId> --prop maxItems --value <n> --json
cai node set <loopNodeId> --prop continueOnIterationError --value true --json
```

MCP: use `node_prop_set` for both writes.

This is not `cai node continue-on-error`, which configures the node-level Error output — a different feature.

Read every returned warning. A list/item type mismatch means the scaffold is not verified even though it was created. Follow the returned `nextSteps` in order.

Inside loop mappings, `item` exists only where context exposes it. Inside the body flow, use its `flow.<itemInput>` binding. Downstream loop results are iteration records whose declared child outputs are under `.outputs`.

Loops run sequentially. `--continue-on-error` applies to a failed child iteration; a list expression or mapping/config expression that fails before the child starts still aborts the loop. Test empty, one-item, several-item, over-limit, and intended iteration-error behavior.

Never enable continue-on-error on a money path; charges, purchases, transfers, refunds, and other spend must fail fast.

## Subflows and convergence

Define the called flow's inputs and outputs first. Then initialize a Run flow component and bind it:

```bash
cai node set-flow <runFlowNodeId> --flow <calledFlowId> --json
cai node props <runFlowNodeId> --json
```

MCP: `cai node set-flow` maps to `node_subflow_set`.

Setting the target flow creates mapping props from the target's current inputs. If those inputs later change, run `set-flow` again and reconfigure mappings.

When paths converge, inspect the concrete graph and execution behavior. Do not assume wiring two paths to one action creates a barrier or waits for both. If the requirement is a true join, discover a join-capable component and verify it with representative branch timing.

Return semantics: a flow that should return data needs a Return response node — without one it returns nothing, and callers (including agents invoking it as a tool) see silent success with empty output. If several Return response nodes complete in one execution, their result objects merge in execution order and a later node silently overwrites an earlier value for the same field. Design for exactly one Return per path unless merging is the intent.

Deleting a flow is a normal, undoable edit: it removes the flow's nodes, edges, inputs/outputs, and attached triggers, and unlinks run-subflow callers in other flows — the same way deleting a function breaks its call sites. No confirmation ceremony is needed; run `cai workflow validate` afterwards to see every break, fix or remove the callers it lists, and `cai workflow restore` if the deletion was a mistake. Turning `continueOnError` off is the same class: it deletes the node's Error output and every edge from it — validate after, restore if regretted.

Read [references/patterns.md](references/patterns.md) for concise canonical patterns.

## Completion gate

- Every branch has the intended condition and a wired target.
- If/Else ordering matches priority and Else remains last.
- Router fan-out is intentional when more than one condition can be true.
- Loop item type and list expression compile against actual source data.
- Expected true, false/Else, empty-list, one-item, several-item, over-limit, and error paths are tested.
- `cai exec tree <workflowExecutionId>` contains no unexplained skipped or missing nodes.
