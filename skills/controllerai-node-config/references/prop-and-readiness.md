<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Prop and readiness reference

## What `node_status` reports

- `missingConnection`: an authenticated component has no account bound.
- `missingRequiredProps`: materialized required props without a value.
- `disabledRequiredProps`: names of required props currently disabled by an activating selector. `blockedBy` and `next` name that selector.
- `pendingReloadProps`: selectors that must be set before dependent props are trustworthy.
- `dynamicSelectorNotes`: reload selectors holding dynamic values whose dependents were not re-resolved. Non-blocking; read them even when the node is ready.
- `outputTypeStatus`: `concrete`, `unresolved` or `none`. Connection-port wrappers never count as a concrete process output.
- `readyForExpressions`: connection and required/reload props complete. The source still has to be wired before its binding appears in expression context.
- `readyForRun`: runnable; the run may be what discovers the output type.
- `readyForDownstreamExpressions`: runnable and the process output type is concrete enough for downstream field bindings. Require this, not `readyForRun`, before binding downstream fields.
- `next`: diagnostic guidance for the next step, phrased as a command line; apply it with the matching typed tool.

## Prop channels

`node_prop_set` picks the channel from live metadata: `value` (or `valueJson` for JSON text kept verbatim) is a literal; `js` is an expression, read back as `valueJs` for text, JSON text and custom data, or `expressionValueJs` for a non-text prop in expression mode; `rawExpressionJson` round-trips a compiled tree. Pass exactly one of the four.

Never force a channel through internal fields. The expression setters compile before writing (a raw tree is validated by the backend), and every setter verifies the actual channel in the readback.

## Option-backed props

Static options are declared in the component metadata. Remote options need the current connection and the earlier prop values. When dependency metadata is present, each option-bearing prop reports:

- `available`: no earlier selector blocks the lookup (a connection or provider failure can still stop it).
- `blocked_upstream` with `blockedBy`: an earlier single-choice selector is unset. Follow the error's exact command first.
- `dynamic_upstream`: an earlier single-choice selector holds an expression, so no concrete option list exists. Write a known exact ID with its display label, or an expression. This is the only case that skips membership validation; it still needs prop metadata and verified readback.

Older backends and responses without a dependency catalog omit these fields; absence means unknown.

Pass the value and the label from the same live option. Many providers search labels rather than opaque IDs, so a copied ID without its label may be impossible to re-verify. Multi-select takes an ordered array of values and an aligned array of labels; the readback shows `{value, label}` records.

`node_prop_options` takes `query`, an opaque `prevContext` from the previous page, and `for` as an array of `"<prop>=<value>"` overrides for a read-only preview under uncommitted earlier values; `node_prop_set` takes the chosen `value` (or an array of values) and `label` as an array of the matching labels.

## Reload selectors

A prop with `reloadProps: true` changes the schema of later props. Set it, then immediately re-read props and status. Provider, model family, spreadsheet, database and action selectors commonly behave this way; never batch their dependents against stale metadata.

A prop reload re-resolves current values without re-setting a selector. If a materialized reload selector is unset, it returns `reconcileSkipped: "unset_selector"` and preserves the materialized dependents instead of reconciling them.

For a component whose dependent schema exists only after a concrete selection: set one concrete value, re-read and configure the revealed dependents, then switch the selector to an expression. The dependents persist from the last concrete selection; starting dynamic leaves them absent, and neither the dynamic write nor a reload materializes them.

## Output type

Resolve an unresolved output by setting the component's supported output or data-type selector, by running the configured node with representative inputs, or by declaring a documented provider sample as unverified. A discovery run (`controllerai-runs`) probes a component that is not in the graph on a scratch node: it deletes that node afterwards, keeps the scratch flow for reuse, and never types a node in the real flow.

A completed run passes its exact node-execution ID back for ownership, workflow, node, component-snapshot and terminal-state checks, then the persisted output shape is adopted for that non-static node only. Read `outputTypeAdoption` (or `outputTypeAdoptions`) in the run result: `no_observed_output` means the run produced no reusable output, which is expected for a sink or control node and, for a data-producing action, a reason to rerun with representative data only when a downstream binding needs the type; `error` means stop and surface it. After adoption, re-read `node_status` and the expression context. An example response alone never establishes a type.
