# Controller AI node configuration


A node is ready only after its component, connection, dependency-ordered props, wiring, and output type are valid. Treat configuration as a state machine, not a bag of property writes.

## Deterministic sequence

### 1. Find an exact component

```bash
cai integration actions "<capability>" --json
cai integration action --integration <nodeSlug> --key <nodeKey> --json
```

Use the returned `nodeSlug` and full `key`. Do not infer a component key from its display name. If discovery is broad, narrow it with:

```bash
cai integration search "<integration>" --actions --json
cai integration actions "<action>" --integration <nodeSlug> --json
```

Inspect static prop metadata before creating the node. `required`, `remoteOptions`, and `reloadProps` determine configuration order.

### 2. Resolve the account

If the component requires auth:

```bash
cai connection list --node <nodeSlug> --json
```

- One active connection: use its exact `id` as the `connectionId`.
- Several: use an explicit user preference; otherwise ask which account.
- None: add the real component without `--connection`; `missingConnection: true` is a valid handoff state. On CLI, get the link with `cai connection connect <nodeSlug> --no-open` and share it right away; do not block on `--wait` before sharing.

  MCP: call `connection_connect_url` with `nodeSlug`, show its `url` as a markdown link, ask the user to finish, then confirm with `connection_list` using its `integration`.

  After they confirm, run `cai connection list --integration <nodeSlug> --json` and bind the new ID with `cai node set-connection <nodeId> --connection <connectionId>`.

Never substitute a connection from a similar slug.

Rename or change the default with `cai connection update <id> [--name "<displayName>"] [--default|--no-default]` (MCP: `connection_update`); delete only intentionally with `cai connection delete <id> --yes` (MCP: `connection_delete`).
Deleting a connection immediately removes authorization from bound nodes and agent tools.

A missing connection is never a reason to substitute `registry_http-http-request` for a catalog component. A raw endpoint mentioned by the user is background context, not an instruction to hand-roll the call.

### 3. Initialize once

```bash
cai node init \
  --flow <flowId> \
  --node <nodeSlug> \
  --key <nodeKey> \
  --name "<data this node produces>" \
  --connection <connectionId> \
  --after <upstreamNodeId> \
  --json
```

MCP: use `node_add`; `cai node init` is the CLI alias of `cai node add`.

Omit flags that do not apply. `--after` creates the ordinary main/error wire at creation. Use it when later props need upstream data. Initialization also accepts `--props`, but use it only for static, non-option values already present in the inspected component metadata. Never inline a `remoteOptions` value, a reload selector and its dependents, or a prop that has not materialized yet; initialize first and set those sequentially. Use `--enable-prop <name...>` only for optional props confirmed by component metadata.

Capture the returned `nodeId`; do not rediscover it by name. Inspect the complete create result before projecting fields. A missing projection is not proof that creation failed, so read the captured ID or exact flow inventory before any retry.

When repairing an existing node, keep its node ID and configure it in place. Do not delete and recreate it merely to clear bad props, select a component, or refresh generated props; deletion discards wiring, conditions, and identity-based references. Replace it only when the requested component itself must change. Then capture the replacement ID, reconnect every dependent, and revalidate them.

### 4. Follow readiness, not guesses

```bash
cai node status <nodeId> --json
cai node props <nodeId> --json
```

For option-backed props, `node props` and write readback may report `optionsAvailability`: `available`, `blocked_upstream` with `blockedBy`, or `dynamic_upstream`. It is positional: earlier single `DROPDOWN` selectors gate later options. Older backends, and responses without a dependency catalog, may omit it; treat absence as unknown, not available.

Work through blockers in this order:

1. Missing connection
2. Required selector props with `reloadProps`
3. Newly revealed required props
4. Static and remote-option props
5. Wiring required for expressions
6. Expressions
7. Unresolved output type

Follow `next`. When a disabled required prop has an activating selector, its blocker names that selector; when it is the next blocker, `next` gives the exact `cai node set` command. `dynamicSelectorNotes` is non-blocking: a node may read ready while its dynamic selector's dependents still reflect the last concrete resolution.

After every concrete reload-selector or connection change, re-run `node props`; dynamic props can appear, disappear, or reset. Use `cai node reload <nodeId> --json` to re-resolve without re-setting a selector. If an explicit reload finds an unset materialized reload selector, it returns `reconcileSkipped: "unset_selector"` plus a note and preserves materialized dependent props instead of reconciling them.

### 5. Set props safely

Static value:

```bash
cai node set <nodeId> --prop <exactName> --value '<literal-or-json>' --json
```

MCP: use `node_prop_set` with `value`; it also handles expression writes through `js`.

Optional prop:

```bash
cai node optional-props <nodeId> --json
cai node set <nodeId> --prop <exactName> --value '<literal-or-json>' --json
# or: cai expr set <nodeId> --prop <exactName> --js '<expression>' --json
```

MCP: `cai expr set` is the `js` form of `node_prop_set`.

`node set` and `expr set` auto-enable a catalog-only optional prop, refresh its metadata, then write. Do not run a separate enable step first. If validation or the later write fails, the prop stays enabled; repair with `cai node disable-prop <nodeId> --prop <exactName>` and re-read props. This is also the repair for the Update-data wipe footgun: that action writes every enabled field, so disable any accidentally enabled field before running it.

MCP: use `node_optional_prop_set` with `enabled: false` to disable or `enabled: true` to enable.

Remote or dynamic option:

```bash
cai node options <nodeId> --prop <exactName> --query "<specific label>" --json
cai node set <nodeId> \
  --prop <exactName> \
  --value '<returned value>' \
  --label '<returned label>' \
  --json
```

`node options` also supports `--prev-context '<context JSON>'` from the preceding page and repeatable `--for '<earlierProp>=<value>'` overrides for read-only hypothetical resolution; JSON literals are accepted and bare text stays a string. If options are `blocked_upstream`, both `node options` and `node set` name the blocking prop and print the exact `cai node set` command to run first.

For an ordinary remote option, preserve both fields from the same returned option. The label is not cosmetic: it is used to search providers that do not index opaque IDs, then the returned value is verified before writing. For a multi-select, pass a JSON array of returned values and one `--label` argument per value, in the same order. The write supplies the persisted label shape, chooses the correct expression storage channel, and verifies read-back.

When an earlier selector makes the prop `dynamic_upstream`, no option list exists. Use a known exact ID or an expression:

```bash
cai node set <nodeId> --prop <exactName> --value '<exact-id>' --label '<display label>' --json
cai node set <nodeId> --prop <exactName> --js '<expression>' --json
```

`--value` stores the literal without provider-option validation and prints a note; on a label-bearing selector, `--label` controls its display label. This is the sole raw `--value` exception when the CLI has an advertised option set: outside `dynamic_upstream`, a missing prop row, a failed or empty advertised remote-options lookup, or a value absent from advertised options fails closed. The exception still requires prop metadata and verified read-back. Do not bypass failures with a raw API call.

For components whose dependent props exist only after a concrete selector value, use a template first: set one concrete value, re-read `node props`, configure or record the revealed dependents, then switch the selector to `--js`. Those materialized props persist as the last concrete schema. Starting dynamic leaves them absent; neither that write nor `node reload` materializes them until one concrete selection is made.

For dynamic values, load `controllerai-expressions` and prefer `cai expr set`.

### 6. Wire and verify

Ordinary outputs:

```bash
cai node connect --flow <flowId> --from <sourceNodeId> --to <targetNodeId> --output main --json
```

Use `--output error` only for the automatic error path. Router and If/Else outputs require `cai branch connect` from the flow-control skill.

Finish with:

```bash
cai node status <nodeId> --json
cai run node <nodeId> --inputs <values.json> --json
```

MCP: `run_node` takes the parsed `inputs` object and `waitSeconds` (default 60, max 120).

If the node has a wired input or consumes `inputs.<name>`, the node test is only an isolated configuration check. Run `cai run flow <flowId> --inputs <values.json> --json` before declaring its upstream expressions verified.

MCP: `run_flow` takes the parsed `inputs` object and `waitSeconds` (default 60, max 120).

If a dynamic provider action cannot be run but its response is documented, declare an unverified output contract:

```bash
cai node declare-output <nodeId> --sample-file provider-response.json --json
```

MCP: call `node_output_type_declare` with the parsed provider response in `sample`; the server cannot read `provider-response.json`.

The sample must be a JSON object or a non-empty JSON array of provider items. Re-read status and expression context afterward; the first real completed run verifies the declaration.

Read [prop-and-readiness.md](prop-and-readiness.md) when a prop is dynamic, option-backed, expression-capable, or unexpectedly absent.

## Hard rules

- Configure props in returned order; later props may depend on earlier values.
- Never write a prop absent from `node props`.
- Let `node set` or `expr set` auto-enable catalog-only optional props; disable the prop explicitly after a failed write.
- Wire before authoring an expression that consumes upstream data.
- Changing a connection invalidates connection-scoped options; re-read props and options.
- A successful write is not completion. Require read-back plus `node status`.
- If output type remains unresolved, perform a realistic development test and inspect the resulting execution. Declare only a documented provider sample; never fabricate an output schema.
