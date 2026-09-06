---
name: controllerai-expressions
description: Write and debug Controller AI typed expressions for flow inputs, upstream outputs, records, conditions and calculations. Use for data mapping, search, filtering, formatting, optional inputs and computed node props.
---

# Expressions: typed values inside a workflow

Expressions connect workflow data to node properties and conditions. The JavaScript-looking authoring language compiles to Controller AI's stored expression tree; it is a **typed subset**, not arbitrary JavaScript execution. You can navigate inputs and fields, calculate numbers, format text and dates, query workflow data and operate on lists.

## Discover the real scope

`expr_operation_list` provides the method catalog, including before a workflow has nodes. `expr_context` for a node shows current flow/input bindings, available fields, upstream outputs not yet wired, and the target property's expected type. Use returned names; display labels and stored IDs are not interchangeable.

Common sources are `flow.<binding>`, `inputs.<binding>`, `process.data`, loop `item`, `now`, `isLiveVersion` and `workflowData("custom.type")`. Upstream values appear under `inputs` only after the relevant output is wired in. Root routing nodes can use flow inputs directly.

## Compose values

```javascript
flow.name.toLowerCase()
`Hello ${flow.name}, score ${flow.score}`
flow.score + 10
flow.label ?? "Unknown"
flow.enabled ? "Enabled" : "Disabled"
workflowData("custom.contact").filter(r => r.id === flow.recordId).first()
```

Use field IDs with the provided dot/bracket spelling. Prefer templates for text composed from multiple values. Dates use the supported date methods and Moment format tokens, for example `now.format("YYYY-MM-DD")`; do not assume JavaScript Date constructors/getters exist.

A record query returns `list.dataRecord.custom.*`, including record identity. Plain custom values describe shape only. Optional filter semantics, membership and pagination are explained in `controllerai-data`.

## Conditions and list predicates

A single comparison or supported boolean expression can be a condition. Router/only-when groups support OR-of-AND groups through the structured condition input. Use that input for compound routing logic instead of assuming every JavaScript boolean expression is accepted everywhere.

List `.filter`, `.some`, `.find` and `.every` accept the predicate grammar in the catalog. `.map` is field projection, not an arbitrary object-building callback. List membership compares an element; text membership finds a case-sensitive substring.

Scalar text/number/boolean ternaries are supported; list, date and record branches need typed routing or a field-mask update instead. Do not assume methods on literals, arbitrary null literals, JavaScript constructors or object-building callbacks are accepted.

If the compiler rejects an idiom, use the error's supported alternative and validate it. Do not invent raw expression trees to bypass a missing language feature. Use routing, typed inputs or a native action where they express the operation more clearly.

## Save and verify

1. Inspect the property and its expected type.
2. Validate the proposed expression with `expr_validate` against the node scope.
3. Persist with `node_prop_set` using `js` (or the relevant condition tool). Literal `value` and compiled `js` are different sources.
4. Read the stored property and resolved status; run representative inputs.
5. Check resolved configs and actual results, including omission, `false`, `0`, empty text/list and missing record cases relevant to the task.

A successful compile proves grammar and type compatibility, not that a query finds data. An intentional empty result can be valid; an unexpected empty value needs tracing to the binding, environment or filter.

Reference syntax and mapping commands: [command reference](references/command-reference.md). Supported grammar and recipes: [DSL reference](references/dsl-and-recipes.md).
