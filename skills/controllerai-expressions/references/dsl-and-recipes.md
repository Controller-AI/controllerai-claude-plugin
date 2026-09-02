# Expression DSL and recipes

The live compiler and `cai expr context`/`cai expr ops` are authoritative. These examples show structure, not names to copy blindly.

## Text template

```js
`Hello ${flow.firstName.trim()}, order ${inputs.order.id} is ready`
```

Use the exact bindings and fields returned for the destination node.

## HTTP JSON body

An HTTP node's JSON/body prop is stored as JSON text. The expression language does not support object literals or `JSON.stringify`. Build the document as a template and insert complete JSON tokens:

```js
`{"message":${flow.name.jsonEncode()},"payload":${flow.user.stringify()}}`
```

- Use `.jsonEncode()` for text or boolean values. It returns a JSON-safe token, including the quotes around text, so do not add another pair of quotes around `${...}`.
- Use `.stringify()` for a custom object or list.
- Keep JSON punctuation outside the interpolation and validate against the destination's exact prop before writing.

```bash
cai expr validate <httpNodeId> --prop <jsonBodyProp> --js-file body-expression.js --json
cai expr set <httpNodeId> --prop <jsonBodyProp> --js-file body-expression.js --json
```

MCP: pass the file contents inline as `js`; the set call uses `node_prop_set`.

## Same-type fallback

```js
flow.count ?? 0
flow.label ?? "unknown"
flow.enabled ?? false
```

Fallbacks must be same-type literals. `0` and `false` remain real values.

## Boolean condition

```js
flow.email.isNotEmpty()
inputs.score >= 80
```

Validate conditions with `--mode onlyWhen` and without a prop name.

## Dates

```js
inputs.createdAt.toDate() > now.plusDays(-7)
inputs.createdAt.toDate().format("YYYY-MM-DD")
```

Convert text to a date before date math. Use negative amounts to move backward.

## Workflow records

Filter immediately after `workflowData` so the engine can push supported predicates into storage:

```js
workflowData("custom.lead")
  .filter(row => row.status === "active")
  .sortBy("created_at", "desc")
```

The collection must exist on the current dev/live rail. A schema declaration alone does not guarantee active records.

## Optional filter inputs

Filters are fail-closed by default: a missing right-hand value matches no records. Use optional matching only when the user's requirement explicitly allows an omitted constraint:

```js
workflowData("custom.lead").filter(
  row => row.ownerId === flow.ownerId,
  { ignoreEmptyConstraints: true }
)
```

Test both the supplied and omitted cases. Empty text may behave as a real value rather than a missing value; do not assume the opt-in drops it.

## Loop item

Where context exposes `item`, primitive lists use the item directly:

```js
item
```

Object lists use an advertised field:

```js
item.email
```

Inside the child flow, the item arrives through a flow input and is referenced as `flow.<name>`, not `item`.

## Loop output

Run-on-list output is a list of iteration records. Child flow outputs live under `outputs`:

```js
inputs.enrichEach[0].outputs.score
```

Confirm the exact binding and field path in context.

## Troubleshooting sequence

1. `cai expr context <node> --prop <prop> --json`
2. Confirm the intended source is wired and its `sourceNodeId` is correct.
3. `cai expr validate ...`
4. `cai expr set ...`
5. `cai expr decompile ...`
6. `cai run node ... --json` for isolated destination configuration
7. `cai run flow <flowId> ... --json` when the expression consumes a wired upstream input
8. `cai exec ports <nodeExecutionId> --json`

MCP: step 4 uses `node_prop_set`; pass expression source and run inputs inline, and set `waitSeconds` on `run_node` or `run_flow` from 0 through 120.

If compilation succeeds but runtime is null or partially empty, inspect `runScope`, the actual input value, and `resolvedConfigs`; do not keep rewriting syntax blindly. A node-only run cannot prove an `inputs.<name>` binding because it does not execute upstream nodes.
