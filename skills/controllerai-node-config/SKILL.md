---
name: controllerai-node-config
description: Discover and configure Controller AI workflow action nodes, dynamic properties, connections, optional fields, output types and graph wiring. Use when assembling or repairing a concrete step in a flow.
---

# Workflow nodes

A node is a step in a flow: a ready integration action, native data/file/AI action, or a control-flow construct. Its component defines properties, required authorization and output shape. The saved node is an instance configured for this workflow.

## Discover by capability

Use `integration_search` and `integration_action_list` to search for the intended action, then inspect the exact integration slug and action key. Integration display names are not component identities. For native storage use `registry_data`; for generated files inspect native file actions; for web research inspect available search actions and connections.

`integration_action_get` describes static properties. Some actions add dependent properties after a type, object, sheet or other selector is chosen. Saved connection status indicates an available binding; a successful representative provider request is the evidence it still works.

## Configure in dependency order

1. Add the action to the intended flow using its returned exact key. Name the node after its purpose or produced value.
2. Bind a required connection. Use `connection_connect_url` only when the user needs to connect an account; never request provider credentials in chat.
3. Set selectors first, reload properties with `node_prop_list`, then read the resulting schema.
4. Enable optional fields before setting them. Omission and an enabled empty value can have different effects, especially for updates.
5. Set literal properties with `value`; use `js` for computed values. Read `expr_context` for expected types and available inputs.
6. Wire the source output channel to the destination. Use returned output/port IDs; “main” and “error” are distinct channels.
7. Inspect `node_status` and `node_get`, test the step or complete flow, then verify the saved output type and downstream mapping.

Use the supported initial configuration batch when creating a node: `node_add` accepts its name, connection, upstream wiring, optional fields and a `props` map together. Direct values are literals; an advanced entry such as `"name": {"valueJs": "flow.name"}` compiles a typed expression. Put selectors such as `dataType` first so dependent fields exist. Inspect the returned `propsReadback` and readiness, then fix only rejected or missing fields. This avoids one call per property when repeating a proven CRUD pattern. Use flow scaffolds for loops and return mappings where they fit.

A static output contract should remain stable after a run. A dynamic action may learn its output shape from a representative test; re-read its output/context before authoring downstream fields. Do not overwrite a known contract with an unrelated inferred schema just to remove a warning.

Use `controllerai-flow-control` for branches/loops and `controllerai-ai-nodes` for AI output configuration. Layout once after a group of structural changes.

Detailed property-setting examples: [command reference](references/command-reference.md). Readiness and dynamic-property behavior: [prop reference](references/prop-and-readiness.md).
