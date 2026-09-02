# Prop and readiness reference

## What `cai node status` means

The JSON response reports:

- `missingConnection`: an authenticated component has no account bound.
- `missingRequiredProps`: materialized required props without a value.
- `disabledRequiredProps`: required props currently disabled by an activating selector. When that selector can be resolved, its blocker names the selector; when it is next, `next` gives the exact `cai node set` command.
- `pendingReloadProps`: selectors that must be set before dependent props are trustworthy.
- `dynamicSelectorNotes`: reload selectors holding dynamic values whose dependents were not re-resolved. These notes do not block readiness, so inspect them even when the node reads ready.
- `outputTypeStatus`: `concrete`, `unresolved`, or `none`. Connection-port wrappers never count as a concrete process output.
- `readyForExpressions`: connection and required/reload props are complete. The source still must be wired before its binding appears in expression context.
- `readyForRun`: connection and required/reload props are complete; running may be the step that discovers a concrete output type.
- `readyForDownstreamExpressions`: the node is runnable and its actual process output type is concrete enough for downstream field bindings.
- `next`: the canonical next diagnostic command.

Follow `next`, then read status again. Do not treat `readyForRun` as proof that downstream fields are typed; require `readyForDownstreamExpressions` for that.

For an existing node with a missing connection, a browser handoff alone is not enough. After the user connects, list the compatible accounts, bind the chosen ID, and re-read status:

```bash
cai connection connect <nodeSlug>
cai connection list --node <nodeSlug> --json
cai node set-connection <nodeId> --connection <connectionId> --json
cai node status <nodeId> --json
```

MCP: replace `cai connection connect` with `connection_connect_url { nodeSlug }`; the other calls map directly.

## Prop channels

Use `cai node set`; it selects the persisted shape from live metadata.

| Intent | Command input | Persisted channel |
| --- | --- | --- |
| Static text, JSON text, scalar, object, or list | `--value` | `value` |
| Dynamic text, JSON text, or custom data | `--js` | `valueJs` |
| Dynamic non-text prop with expression mode | `--js` | `expressionValueJs` |

MCP: both `cai node set` and `cai expr set` use `node_prop_set`; pass exactly one of `value`, `js`, or `rawExpressionJson`.

Do not force a channel with internal fields. `cai node set` and `cai expr set` compile first and verify the actual channel after writing.

## Option-backed props

Static options are declared in component/prop metadata. Remote options require current connection and earlier prop values.

When dependency metadata is available, each option-bearing prop reports:

- `available`: no earlier selector blocks option resolution; connection or provider failures can still stop the lookup.
- `blocked_upstream` with `blockedBy`: an earlier single `DROPDOWN` selector is unset.
- `dynamic_upstream`: an earlier single `DROPDOWN` selector is dynamic, so no concrete option list exists.

The backend computes this in prop order. Older backends, and current responses without a dependency catalog, omit these fields; do not interpret absence as `available`.

```bash
cai node options <nodeId> --prop <name> --query "<specific search>" --json
cai node set <nodeId> --prop <name> --value '<option.value>' --label '<option.label>' --json
```

Pass the value and label from the same live option. Remote providers often search labels rather than opaque values, so omitting `--label` can make a valid copied ID impossible to re-verify. For multi-select, pass a JSON array of returned values plus one `--label` argument per value in matching order. The write constructs the required array of `{value,label}` records for read-back/UI compatibility.

Continue a provider page with the opaque context printed by the previous call:

```bash
cai node options <nodeId> --prop <name> --prev-context '<context JSON>' --json
```

Preview options under uncommitted earlier values with repeatable overrides; valid JSON becomes a typed literal and bare text remains a string:

```bash
cai node options <nodeId> --prop <name> --for '<selector>=<value>' --for '<other>=true' --json
```

For `blocked_upstream`, follow the error's exact command for `blockedBy`. For `dynamic_upstream`, `node options` returns no list; `node set --value '<exact-id>'` deliberately stores a known literal without provider membership validation and prints a note. Add `--label '<display label>'` when the prop is label-bearing, or use `node set --js '<expression>'`. For an advertised option set outside that state, a missing prop row, a failed or empty advertised remote-options lookup, or a value absent from advertised options fails closed.

## Reload selectors

A prop with `reloadProps:true` changes the downstream prop schema. Set it statically, wait for the mutation, and immediately re-read:

```bash
cai node set <nodeId> --prop <selector> --value '<value>' --json
cai node props <nodeId> --json
cai node status <nodeId> --json
```

Use `cai node reload <nodeId> --json` to re-resolve current values explicitly instead of re-setting a selector. If any materialized reload selector is unset, the command returns `reconcileSkipped: "unset_selector"` and a note; that skipped path updates metadata without reconciling or deleting materialized dependent props.

Do not batch dependent values into stale metadata. A provider, model family, spreadsheet, database, or action selector commonly triggers this behavior.

For a component whose dependent schema exists only after a concrete upstream selection, materialize a template before going dynamic:

1. Set the selector to one concrete value.
2. Re-read props and configure or record the revealed dependents.
3. Switch the selector to a dynamic value with `--js`.

The dependent props persist from the last concrete selection. Starting with a dynamic selector leaves them absent, and neither the dynamic write nor an explicit reload can materialize them until the selector has held a concrete value once.

## Output type

An unresolved output blocks trustworthy downstream field expressions. Resolve it by one of:

- Set the component's supported output/data type selector.
- Run the configured node with representative inputs.
- Probe an action with `cai run discover` only when its real-world side effects are acceptable.

After the run, read `cai node status` and `cai expr context` again. Do not infer fields from an example response alone if the platform did not adopt a concrete type.

Completed node and flow runs pass the exact node-execution ID back for ownership, workflow, node, component-snapshot, and terminal-status checks before deriving the persisted output shape; callers never supply the type name. Adoption updates only that non-static node, so separate instances of generic components such as HTTP can learn different schemas. Inspect `outputTypeAdoption`/`outputTypeAdoptions` in the run result. If it reports `no_observed_output`, use a representative input that returns data and rerun. If it reports `error`, stop and surface that error rather than inventing a schema.

MCP: `run_node` and `run_flow` take `inputs` inline and `waitSeconds` from 0 through 120.
