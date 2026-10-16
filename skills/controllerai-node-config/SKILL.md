---
name: controllerai-node-config
description: Discover, add, connect, configure, wire, and verify Controller AI workflow nodes. Use when adding an action, choosing a component, binding an account, setting props (static, optional, option-backed or dynamic), wiring nodes, resolving an output type, or fixing a node that is not ready.
---
<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->

# Node configuration

## Use this when

- A flow needs an action from an app or a native service, and the node has to be found, added, bound to an account, configured and wired.
- A node reports missing connection, missing or disabled required props, pending selectors, or an unresolved output type.

Computed values and conditions belong to `controllerai-expressions`; Router, If/Else, loops and subflows to `controllerai-flow-control`; Ask AI provider, model and typed output to `controllerai-ai-nodes`; tests and execution evidence to `controllerai-runs`.

## How it works

A **node** is a configured instance of a **component**: an integration action, a native data, file or AI action, or a control node. The component defines the props, whether an account is required, and the output shape. A node becomes usable in stages: component, connection, props in dependency order, then wiring and a concrete output type for whatever reads it downstream; each stage has its own readiness signal, so configure it as a sequence, not as one batch of property writes.

**Connections** are the authorized accounts of an integration. A saved active connection proves a binding exists, not that the provider still accepts it; a representative request proves that. Changing a node's connection invalidates every connection-scoped option: re-fetch and re-verify them.

**Props** come in kinds: static values; optional props that exist in the catalog and appear once enabled; option-backed props whose values come from static lists or a provider lookup; reload selectors (`reloadProps: true`) that change the schema of later props; and expression-capable props that take a dynamic value. Options are resolved in prop order: an earlier single-choice selector gates later lookups. `optionsAvailability` reports `available`, `blocked_upstream` with `blockedBy`, or `dynamic_upstream`; when the field is absent, availability is unknown, not available.

**Readiness** (the node status read) reports distinct blockers: `missingConnection`, `missingRequiredProps`, `disabledRequiredProps` (the selector that activates them is named in `blockedBy` and `next`), `pendingReloadProps`, an unset subflow target or an invalid static option, and `dynamicSelectorNotes` (non-blocking: dependents still reflect the last concrete selection). `readyForExpressions` means the connection and the required and reload props are complete; `readyForRun` means it can execute; `readyForDownstreamExpressions` additionally means the output type is concrete. Readiness does not check wiring. `outputTypeStatus` is `concrete`, `unresolved` or `none` (a sink or control node has no output); an unresolved integration output is learned from a representative run, a supported output selector or a declared sample, and adoption changes only that node, so two instances of a generic component can learn different shapes. `next` is diagnostic guidance phrased as a command line: apply it with the matching typed tool rather than executing it blindly.

## Rules

1. **Use the exact component the discovery returned** (`nodeSlug` and full `key`); never infer a key from a display name, and never swap in a similar integration or a raw HTTP request (`registry_http-http-request`) because a connection is missing. A user's mention of an endpoint is context, not a request to hand-roll the call.
2. **Add a node without a connection when no account exists.** `missingConnection: true` is a valid handoff state; get the user's authorization and bind the exact new connection ID afterwards. Never ask for a provider credential in chat.
3. **Configure in dependency order and read back after every write.** Set reload selectors before their dependents, re-read props after each selector or connection change, and treat a write as done only after readback plus `node status`.
4. **Write only props that exist.** Materialized props and catalog optionals are writable; a name absent from both is an error. Setting a catalog optional enables it automatically; do not enable first. If that write then fails, the prop stays enabled: disable it and re-read.
5. **Repair in place.** Keep the node ID, wiring, conditions and references, because downstream bindings and subflow mappings point at that ID. Replace a node only when the requested component itself changes; then reconnect and revalidate every dependent.
6. **Keep value and label together for option-backed props.** Labels are how providers that do not index opaque IDs are searched and how values are verified. Outside `dynamic_upstream`, a value that is not in the advertised options fails closed.
7. **Wire before writing an expression that reads upstream data**, and use branch connections for Router and If/Else outputs.
8. **Never fabricate an output schema.** Resolve the type with a representative run, a supported output selector, or a documented provider sample declared as unverified; preserve a static contract instead of overwriting it with an inferred one.
9. **Update Data writes every enabled field that resolves to a value.** A field that resolves to nothing is left out, but an enabled field that resolves to null or empty text overwrites the stored value, so disable fields you enabled by accident and inspect the resolved values before running.

## Recipes

### Find the component

- `integration_action_list` — `query`, `agentAttachable`, `integration`, `limit`
- `integration_action_get` — `integration`, `key`

Narrow a broad result with `integration_search`, then `integration_action_list`. Read `required`, `remoteOptions` and `reloadProps` before adding: they decide the configuration order.

### Resolve the account

- `connection_list` — `integration`

A row with `shared: true` is a teammate's account shared with the organization, which the user may use. One active connection of the user's own: use its ID. Several: follow the user's stated preference, otherwise ask which account. None of their own but a shared one: name it and ask whether to use it or connect their own; bind it only after a yes. None at all: add the node without a connection and hand off.

Call `connection_connect_url` with the `nodeSlug`, show the returned `url` as a markdown link, ask the user to finish, then `connection_list` for that integration and bind the new ID. A URL is a handoff, not a completed authorization.

- `node_connection_set` — `nodeId`, `connection`, `workflowId`
- `node_status` — `nodeId`, `workflowId`

### Add the node

- `node_add` — `after`, `connection`, `enableProp`, `flow`, `key`, `name`, `node`, `output`, `props`, `workflowId`

Name the node after its purpose or the value it produces. `after` wires the upstream node's main output (`output: "error"` for its error path) at creation, which later expressions need; initial `props` may carry only inspected static values, and `enableProp` lists optionals confirmed by the metadata. Never pass a remote option, a reload selector with its dependents, or a prop that has not materialized at creation. Capture the returned `nodeId`, then inspect `propsReadback` and `readiness` in the result and fix only rejected or missing fields.

### Follow readiness

- `node_status` — `nodeId`, `workflowId`
- `node_prop_list` — `nodeId`, `workflowId`

Clear blockers in this order: connection; required reload selectors; newly revealed required props; static and option-backed props; wiring an expression needs; expressions; output type. Use `next` as the guide for the next step. After a concrete selector or connection change, re-read props: fields appear, disappear or reset. `node_prop_reload` re-resolves dependents without re-setting a selector; when a materialized selector is unset it returns `reconcileSkipped: "unset_selector"` and preserves the dependents.

### Set a prop

- `node_prop_set` — `nodeId`, `js`, `label`, `prop`, `rawExpressionJson`, `value`, `workflowId`, `valueJson`
- `node_optional_prop_list` — `nodeId`, `workflowId`
- `node_prop_options` — `nodeId`, `for`, `page`, `prevContext`, `prop`, `query`, `workflowId`
- `node_prop_set` — `nodeId`, `js`, `label`, `prop`, `rawExpressionJson`, `value`, `workflowId`, `valueJson`
- `node_optional_prop_set` — `nodeId`, `prop`, `workflowId`, `enabled`

`node_prop_set` takes exactly one of `value`, `valueJson` (JSON text kept verbatim), `js` or `rawExpressionJson`; for an option-backed prop pass `label` as an array of the returned labels aligned with the values (a one-element array for a single selection). `node_prop_options` continues a page with `prevContext` (parsed JSON) and previews under uncommitted earlier values with `for` as an array of `"<prop>=<value>"` strings. `node_optional_prop_set` with `enabled: false` disables; `true` enables. When options are `blocked_upstream`, the error names the selector to set first. For `dynamic_upstream`, no list exists: set a known exact ID with its label, or an expression in `js`.

For a component whose dependent props exist only after a concrete selection, materialize a template first: set one concrete value, re-read props and configure the revealed dependents, then switch the selector to an expression. The dependents persist from that last concrete selection; starting dynamic leaves them absent. For dynamic values, load `controllerai-expressions`.

### Wire

- `node_connect` — `flow`, `from`, `output`, `to`, `workflowId`

`output: "error"` is only for the automatic error path. Router and If/Else outputs use branch connections from `controllerai-flow-control`.

### Resolve the output type

- `run_node` — `nodeId`, `inputs`, `waitSeconds`, `workflowId`
- `node_output_type_declare` — `nodeId`, `sample`, `workflowId`

`run_node` takes parsed `inputs` and `waitSeconds` (default 60, maximum 120). `node_output_type_declare` takes the parsed provider response in `sample`; the server cannot read a file path.

Prefer the representative run: the completed execution is checked for ownership and terminal state, then its output shape is adopted for that node. Inspect `outputTypeAdoption` in the run result: `no_observed_output` on a data-producing action means the run returned nothing reusable, so rerun it with inputs that return data only when a downstream binding needs the type and the additional real action is authorized; on a sink or control node it is the expected result. `error` means stop and surface it. Declare a documented sample (an object or a non-empty array of items) when the action is unavailable, unsafe or declined; it stays unverified until the first real run. A node test with a wired input is an isolated check; upstream expressions are verified only by running the containing flow. Adoption is node-local: a discovery run on a scratch node never types the node in the flow. Read `controllerai-runs` for discovery.

## Verify

`node status` shows no blockers for the requested readiness level, every written prop reads back with the intended value and label, the wiring matches the flow, and the output type is concrete before downstream fields are bound. Expression context (`expr_context`) lists the upstream binding you expect.

## Report

The component and connection used, which props were set and which remain for the user (a missing authorization above all), and whether the output type was verified by a run or only declared.

Read [references/prop-and-readiness.md](references/prop-and-readiness.md) when a prop is dynamic, option-backed, expression-capable, or unexpectedly absent, or when a status field needs its exact meaning.
