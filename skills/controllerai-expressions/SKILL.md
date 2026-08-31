---
name: controllerai-expressions
description: Author, validate, persist, and debug Controller AI expressions for node props and conditions. Use when a value comes from a flow input, upstream node, loop item, workflow data, constant, calculation, template, or boolean rule, or when an expression compiles but resolves to null or the wrong input.
---

# Controller AI expressions

In MCP hosts, send each exact literal `cai ...` command through the single `exec` tool; never translate it into another grammar.

Controller AI uses a restricted JavaScript-like authoring language that is compiled into the workflow engine's expression AST. It is not general JavaScript. Discover the real scope and compile before every write.

Data types are the TypeScript types of workflows: every expression is typed, and an expression can only be written against a known type. Integration types describe what integration actions output (resolved by a successful dev run); `custom.*` types describe shapes for data-store records and AI-node structured outputs. Two wrappers matter: `dataRecord.custom.x` means "an actual stored record of that shape" — the fields plus record identity (`id`, `createdAt`) — while bare `custom.x` is just data in that shape; `list.` prefixes a list of any type. When an expression cannot be authored, the cause is usually a missing type — resolve the upstream node's output type first, don't guess field paths.

`.every()` accepts the same multi-condition predicates as `.filter()` and compiles to the
single-pass `:all match` op. A constraint whose field no longer exists in the schema makes
`.every()` false; `.filter()` drops that constraint. Per-row missing, null, and unparseable values
follow the comparator's own semantics, and negative comparisons can pass missing or null rows.
For guard conditions, require field presence explicitly.

## How empty values evaluate

An empty input is not "nothing" — it evaluates to the type's empty form: empty text `""` for TEXT, null/undefined for objects. Two platform semantics follow, and both fail silently rather than erroring:

- **A filter comparing against an empty value still filters — and matches NO records.** This is deliberate (fail-closed; matching everything by accident would be worse). If "empty should mean skip this filter", opt that constraint into ignore-empty mode explicitly (`ignoreEmptyConstraints`, see [references/dsl-and-recipes.md](references/dsl-and-recipes.md)).
- **An optional TEXT input left empty arrives as `""`**, so an `equals` constraint against it matches nothing — the classic symptom is "search finds zero results only when the optional filter is blank", which then cascades (e.g. a create-if-missing step makes duplicates).

When a run unexpectedly returns zero results, check the resolved configs for empty comparison values first (`cai exec ports`).

## Required write loop

### 1. Wire first

An upstream output is available only after it is connected to the target node. If context lists it under `unwiredUpstreamOutputs`, connect it, then fetch context again.

```bash
cai node connect --flow <flowId> --from <sourceNodeId> --to <targetNodeId> --json
```

Use `cai branch connect` for Router or If/Else outputs.

### 2. Read the destination's exact scope

```bash
cai expr context <nodeId> --prop <propName> --json
```

Copy binding and field names exactly. Important sources include:

- `flow.<name>`: flow input
- `inputs.<name>`: wired node input; the payload is usually already unwrapped
- `item`: injected current item, only where context exposes it
- `workflowData("custom.type")`: records from an active workflow collection
- `now`, `isLiveVersion`: constants shown by context

Do not derive a binding from a display label. Context flags duplicate display names and supplies the de-duplicated JS name, input ID, and source node.

When compilation reports an unknown field, copy a typeable display-derived accessor it advertises; field IDs appear second for disambiguation.

### 3. Use only advertised syntax

Get the current operation catalog when needed:

```bash
cai expr ops --type <text|number|date|list|boolean|file|custom> --json
```

Keep the program to optional `const` declarations followed by one final expression. Do not use loops, assignments, `new`, regular expressions, optional chaining, arbitrary library methods, or object construction unless the current compiler catalog explicitly supports them.

Date values are ISO-8601 strings on the wire (`"2026-06-05"` or `"2026-06-05T12:30:00Z"`). A `date`-typed flow input or field is used directly with date operations; only `text` values need `.toDate()` first. For epoch text, pass the unit literal explicitly: `flow.timestamp.toDate("epoch_seconds")` or `flow.timestamp.toDate("epoch_millis")`. The ops table renders that argument generically; never guess the unit from digit count.

Read [references/dsl-and-recipes.md](references/dsl-and-recipes.md) for canonical patterns.

### 4. Compile, write, and read back

For a node prop, prefer:

```bash
cai expr set <nodeId> --prop <propName> --js '<expression>' --json
```

For multiline expressions or shell-sensitive templates, put the exact source in a UTF-8 file and use `--js-file <path>`. Use `--js-file -` to read stdin. It is mutually exclusive with inline `--js` and is available on `expr validate/set`, `node set/only-when`, and `branch add/update`.

```bash
cai expr set <nodeId> --prop <propName> --js-file expression.js --json
```

This fetches target metadata, auto-enables a catalog-only optional target when needed, compiles before writing the value, persists to the correct channel, and verifies read-back. If validation or the value write then fails, the prop stays enabled; repair with `cai node disable-prop <nodeId> --prop <propName>`.

Array channels are not interchangeable. On a value-channel `SHORT_TEXT_ARRAY`, a scalar expression replaces the whole array with one dynamic element; it does not append. List-typed expressions on value-channel array props are rejected. Use `cai node set <nodeId> --prop <propName> --value '["a","b"]'` for a static list, or use a list-typed expression only where prop metadata exposes whole-array expression mode.

For a side-effect-free check:

```bash
cai expr validate <nodeId> --prop <propName> --js '<expression>' --json
```

For conditions:

```bash
cai expr validate <nodeId> --mode onlyWhen --js '<boolean expression>' --json
cai node only-when <nodeId> --js '<boolean expression>' --json
```

Router and If/Else conditions are written with `cai branch add` or `cai branch update` after the same context/validation discipline.

### 5. Verify runtime resolution

```bash
cai expr decompile <nodeId> --prop <propName> --json
cai run node <nodeId> --inputs <values.json> --json
```

Use `run node` to isolate the destination's own configuration. It never executes wired upstream nodes. If the expression consumes `inputs.<name>`, also run its containing flow:

```bash
cai run flow <flowId> --inputs <values.json> --json
```

Inspect `runScope` and `resolvedConfigs` in the run result. A null or missing dynamic config is an expression failure, not success; a partially empty value is equally suspect. Only the full flow run proves an upstream-derived value. Use `cai exec ports <nodeExecutionId> --json` to compare inputs, resolved configs, and outputs.

## Hard rules

- Never guess a binding, field, method, or target type.
- Never author against an unwired output.
- Never add `.data` automatically. Use the exact path shown by context.
- Re-read context after rewiring, renaming inputs, changing schemas, changing dynamic selectors, or resolving an output type.
- Preserve `0` and `false`; they are values, not missing data.
- Optional filters need an explicit behavior decision. Do not silently change fail-closed filtering to match-all.
- A compiler success proves syntax and types, not correct source selection. Verify the source node/input IDs when names are ambiguous and inspect runtime values.
