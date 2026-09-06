---
name: controllerai-flow-control
description: Build Controller AI execution paths with conditions, routers, if/else branches, loops, joins and subflow calls. Use when a flow must validate, branch, iterate or reuse another flow.
---

# Routing, loops and reusable flows

Control-flow nodes decide which steps execute and how data moves between paths. Use conditions to validate requests, a router for named alternatives, a loop for each item in a list, and subflows for reusable behavior.

## Choose the construct

- **Only when:** skip one node unless a condition holds.
- **If/Else:** two paths, such as record found versus not found.
- **Router:** several named routes, such as search modes or operation types.
- **For each / Run Flow on List:** execute a body flow for each typed item and collect iteration results.
- **Subflow:** call a reusable typed flow and map its inputs/outputs.
- **Join paths:** deliberately reconcile active paths before a shared continuation.

Choose whether multiple matching routes should run or one alternative should win. Make invalid/missing-input paths return a clear result without performing the write. A skipped node is expected only if its condition and port explain why.

## Build branches

Inspect the router's current expression context, define a route condition, and connect the route output to its path. Conditions can use the flow's inputs and wired upstream data. Use the supported structured OR-of-AND condition groups for compound logic; do not assume ordinary JavaScript syntax is valid in every condition surface.

A condition accepting a record requires a fetched record, not a fabricated object. Test both branches, including false/zero values and a missing record. Declare and populate the flow output on each intended return path.

## Build loops

Use the available for-each scaffolding operation to create the loop, body flow, typed item input and mappings together. It reduces the mistakes involved in assembling loop injection manually. Supply the list expression and exact source output when wiring an upstream list.

The body receives one item; it does not receive the whole original list implicitly. Custom-shaped items and real data-record items have different types. Loop output represents iteration results, so inspect its actual fields before mapping a final list.

For an uploaded CSV, a conversational agent can read rows and call a per-row flow. Use a native loop when the workflow already receives a typed list and deterministic iteration is desired.

## Verify paths

Run one representative case per route and inspect node status, selected output ports, skipped reasons and final outputs. For writes, verify the same-environment record effects. A successful outer loop does not prove every iteration succeeded; surface row failures in the result.

Examples and commands: [command reference](references/command-reference.md). Control-flow details: [patterns](references/patterns.md).
