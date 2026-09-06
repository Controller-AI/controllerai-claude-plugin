# Controller AI expressions


Controller AI uses a restricted JavaScript-like authoring language that is compiled into the workflow engine's expression AST. It is not general JavaScript. Discover the real scope and compile before every write.


- Wire first; refresh context after structural changes. See Required write loop.
- `--value` is static; dynamic values use `--js`. See Typed values and templates.
- Copy current bindings, not labels or stale names. See Read the destination's exact scope.
- Never use raw ASTs, general JavaScript, or private APIs. See Use only advertised syntax.
- Keep ternaries in the destination type. See Typed values and templates.
- Prove resolution in a containing-flow run. See Verify runtime resolution.

Data types are the TypeScript types of workflows: every expression is typed, and an expression can only be written against a known type. Integration types describe what integration actions output (resolved by a successful dev run); `custom.*` types describe shapes for data-store records and AI-node structured outputs. Two wrappers matter: `dataRecord.custom.x` means "an actual stored record of that shape" — the fields plus record identity (`id`, `createdAt`) — while bare `custom.x` is just data in that shape; `list.` prefixes a list of any type. When an expression cannot be authored, the cause is usually a missing type — resolve the upstream node's output type first, don't guess field paths.

`.every()` accepts the same multi-condition predicates as `.filter()` and compiles to `:all match`. A removed schema field makes `.every()` false but is dropped by `.filter()`. Comparator semantics govern missing, null, and unparseable values; guards must require presence explicitly.

## How empty values evaluate

An empty input evaluates to its typed empty form: `""` for TEXT, null/undefined for objects. Both cases below fail silently:

- **A filter compared with empty matches no records.** To skip an omitted constraint, opt into `ignoreEmptyConstraints`; see [dsl-and-recipes.md](dsl-and-recipes.md).
- **An omitted optional TEXT input arrives as `""`.** Equality then matches nothing and can make a create-if-missing step duplicate data.

When a run unexpectedly returns zero results, check the resolved configs for empty comparison values first (`cai exec node <nodeExecutionId> --full`).

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
cai expr ops --workflow <workflowId> --type <text|number|date|list|boolean|file|custom> --json
```

The catalog is workflow-scoped because it is read from expression context. On CLI, an existing `cai use` or `CAI_WORKFLOW` selection can replace the explicit flag; MCP `expr_operation_list` always requires `workflowId`.

Keep the program to optional `const` declarations followed by one final expression. Do not submit raw AST JSON or probe private APIs. Do not use loops, assignments, `new`, regular expressions, optional chaining, arbitrary library methods, or object construction unless the compiler catalog supports them.

Date values are ISO-8601 strings on the wire (`"2026-06-05"` or `"2026-06-05T12:30:00Z"`). A `date`-typed flow input or field is used directly with date operations; only `text` values need `.toDate()` first. For epoch text, pass the unit literal explicitly: `flow.timestamp.toDate("epoch_seconds")` or `flow.timestamp.toDate("epoch_millis")`. The ops table renders that argument generically; never guess the unit from digit count.

#### Typed values and templates

`--value` stores a literal, so `flow.name`, `{inputs.Name}`, `{{name}}`, and `${name}` stay static. `--js` and `--js-file` compile typed expressions. `${...}` interpolates only inside a backtick template expression, which returns text. Keep non-text values as direct typed expressions.

Ternaries preserve compatible scalar branch types: number and boolean branches stay number and boolean. A different inferred destination type means the write is incomplete. Report the compiler error; do not quote or coerce branches to text.

Read [dsl-and-recipes.md](dsl-and-recipes.md) for canonical patterns.

### 4. Compile, write, and read back

For a node prop, prefer:

```bash
cai expr set <nodeId> --prop <propName> --js '<expression>' --json
```

MCP: use `node_prop_set` with `js`; it merges `cai expr set` with `cai node set`.

For multiline expressions or shell-sensitive templates on CLI, put the exact source in a UTF-8 file and use `--js-file <path>`. Use `--js-file -` to read stdin. It is mutually exclusive with inline `--js` and is available on `expr validate/set`, `node set/only-when`, and `branch add/update`.

```bash
cai expr set <nodeId> --prop <propName> --js-file expression.js --json
```

MCP: pass the file contents inline as `js`; the hosted process does not read the local path or stdin.

This fetches target metadata, auto-enables a catalog-only optional target when needed, compiles before writing the value, persists to the correct channel, and verifies read-back. If validation or the value write then fails, the prop stays enabled; repair with `cai node disable-prop <nodeId> --prop <propName>`.

MCP: repair with `node_optional_prop_set` and `enabled: false`.

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

Node `only-when` takes one restricted boolean through `--js` or `--js-file`, or grouped conditions through `--groups-json '[["a","b"],["c"]]'` (inner arrays are AND, outer rows are OR). Root scalar `&&` or `||` inside one expression is unsupported; use `--groups-json` for compound guards. Branch add/update accepts `string[][]`: outer rows are OR, inner entries are AND, and each entry compiles separately.

```bash
cai branch update <nodeId> --route <routeId> --condition-json '[["flow.email.isNotEmpty()","flow.units < 5"],["flow.override === true"]]' --json
```

Use grouping only when branch semantics fit. For a compound node-only guard, use `--groups-json`; do not weaken the rule or probe private APIs. To make missing numeric input fail `< 5`, use `(flow.unitsLeft ?? 5) < 5`.

Router and If/Else conditions are written with `cai branch add` or `cai branch update` after the same context/validation discipline.

### 5. Verify runtime resolution

```bash
cai expr decompile <nodeId> --prop <propName> --json
cai run node <nodeId> --inputs <values.json> --json
```

MCP: `run_node` takes the parsed `inputs` object and `waitSeconds` (default 60, max 120).

Use `run node` to isolate the destination's own configuration. It never executes wired upstream nodes. If the expression consumes `inputs.<name>`, also run its containing flow:

```bash
cai run flow <flowId> --inputs <values.json> --json
```

MCP: `run_flow` takes the parsed `inputs` object and `waitSeconds` (default 60, max 120).

Inspect `runScope` and `resolvedConfigs` in the run result. A null or missing dynamic config is an expression failure, not success; a partially empty value is equally suspect. Only the full flow run proves an upstream-derived value. Use `cai exec node <nodeExecutionId> --full --json` to inspect the final status, resolved configs, and untruncated output.

## Hard rules

- Never guess a binding, field, method, or target type.
- Never author against an unwired output.
- Never add `.data` automatically. Use the exact path shown by context.
- Re-read context after rewiring, renaming inputs, changing schemas, changing dynamic selectors, or resolving an output type.
- Preserve `0` and `false`; they are values, not missing data.
- Optional filters need an explicit behavior decision. Do not silently change fail-closed filtering to match-all.
- A compiler success proves syntax and types, not correct source selection. Verify the source node/input IDs when names are ambiguous and inspect runtime values.

## Before you finish

```bash
cai expr context <nodeId> --prop <propName> --json
cai expr validate <nodeId> --prop <propName> --js '<expression>' --json
cai expr decompile <nodeId> --prop <propName> --json
cai run flow <flowId> --inputs <values.json> --json
```

- Context must show current source IDs and bindings.
- Validation and decompile must agree on source and type.
- The flow run must show the intended value in `runScope` and `resolvedConfigs`.
