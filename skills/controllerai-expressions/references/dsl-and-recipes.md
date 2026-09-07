<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Expression DSL and recipes

The live compiler, `cai expr context` and `cai expr ops` are authoritative. These examples show structure; take every binding, field and operation name from the context.

## Text template

```js
`Hello ${flow.firstName.trim()}, order ${inputs.order.id} is ready`
```

## Text prefix

When `.startsWith()` is not advertised, test a prefix with a guarded split:

```js
flow.subject.isNotEmpty() ? flow.subject.split("urgent")[0] === "" : false
```

The guard keeps empty text from matching. Prefix matching is not first-word matching: do not split on whitespace unless first-word equality is the requirement, and test a value with punctuation right after the prefix (`urgent-alpha`) plus a near miss.

## HTTP JSON body

A JSON body prop is JSON text. The language has no constructed object literals and no `JSON.stringify`, so build the document as a template of complete JSON tokens:

```js
`{"message":${flow.name.jsonEncode()},"payload":${flow.user.stringify()}}`
```

`.jsonEncode()` returns a JSON-safe token: quoted for text, bare `true`/`false` for a boolean, so never add a second pair of quotes. `.stringify()` serializes a custom object or list. Keep the JSON punctuation outside the interpolations and validate against the exact body prop before writing.

## Same-type fallback

```js
flow.count ?? 0
flow.label ?? "unknown"
flow.enabled ?? false
```

`0` and `false` are values; a fallback never replaces them.

## Boolean condition

```js
flow.email.isNotEmpty()
inputs.score >= 80
```

Validate a condition with `--mode onlyWhen` and no prop name.

## Dates

```js
inputs.createdAt.toDate() > now.plusDays(-7)
inputs.createdAt.toDate().format("YYYY-MM-DD")
```

Convert text to a date before date math; a negative amount moves backward. Use Moment format tokens, never JavaScript `Date` constructors or getters.

## Workflow records

Filter immediately after `workflowData` so supported predicates are pushed into storage, then sort by an advertised field:

```js
workflowData("custom.lead")
  .filter(row => row.status === "active")
  .sortBy("createdAt", "desc")
```

The collection must exist on the current rail; a schema declaration alone does not put rows in it. Record identity fields are `id`, `createdAt`, `updatedAt`.

## Optional filter inputs

An empty right-hand value is a real comparison: equality matches only empty fields, a negation or containment matches every row. Skip an omitted constraint only when the requirement allows it:

```js
workflowData("custom.lead").filter(
  row => row.ownerId === flow.ownerId,
  { ignoreEmptyConstraints: true }
)
```

With the option on, null, empty text and an empty list skip the constraint; `0`, `false`, whitespace and dates remain real constraints. Test the supplied and the omitted case.

## List membership

For a list field such as tags, membership is a single element: `row.tags.includes(flow.tag)`. `filter`, `some`, `find` and `every` share the same predicate grammar; `map` projects fields rather than building objects.

## Primitive lists

A predicate compares a field of the item, so `flow.weights.filter(w => w >= flow.threshold)` is rejected, and `w.value` on a number compiles as a missing field whose constraint is dropped, matching every item. Aggregate a primitive list with the advertised list operations (`.sum()`, `.count()`, `.max()`), and classify or count its items with a per-item loop whose body returns a flag:

```js
inputs.runFlowOnList.filter(row => row.status === "completed").map(row => row.outputs).map(row => row.heavyFlag).sum()
```

`.map()` picks one field per call, so nested picks are chained; the `status` filter keeps failed iterations out of the total.

## Loop item

Where the context exposes `item`, a primitive list uses it directly:

```js
item
```

An object list uses an advertised field:

```js
item.email
```

Inside the child flow the item arrives as a flow input and is `flow.<name>`, never `item`. A root routing node can read flow inputs directly; no producer node is needed to expose them.

## Loop output

Run-on-list output is a list of iteration records; child outputs sit under `outputs`:

```js
inputs.enrichEach[0].outputs.score
```

Confirm the binding and the path in the context.

## Troubleshooting order

1. `cai expr context <nodeId> --prop <propName> --json`
2. Confirm the intended source is wired and its `sourceNodeId` is right.
3. `cai expr validate <nodeId> --prop <propName> --js '<expression>' --json`
4. `cai expr set <nodeId> --prop <propName> --js '<expression>' --json`
5. `cai expr decompile <nodeId> --prop <propName> --json`
6. `cai run node <nodeId> --inputs <values.json> --json` for the destination alone
7. `cai run flow <flowId> --inputs <values.json> --json` when the expression consumes a wired input
8. `cai exec node <nodeExecutionId> --full --json`

If compilation succeeds but the runtime value is empty or partial, inspect `runScope`, the actual input value and `resolvedConfigs` before touching the syntax again. A node-only run cannot prove an `inputs.<name>` binding because it does not execute upstream nodes.
