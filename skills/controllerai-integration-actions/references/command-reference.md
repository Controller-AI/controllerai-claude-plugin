# Controller AI integration actions


Ready integration actions give a hosted agent one app operation without first building a workflow. Use the canonical operations below through the selected transport; do not call Controller AI REST routes directly.


- Attach one existing app operation directly; never wrap it in a one-node workflow. See Choose direct action or workflow.
- Capture the exact integration, action, connection, and tool IDs. See Discover exact identities.
- Decide confirmation before attachment, especially for writes or spend. See Decide approval before attaching.
- Verify persisted direct-action events, not a workflow execution ID. See Verify through the hosted agent.

## Choose direct action or workflow

Use a direct action for one existing operation such as sending one message, creating one record, or adding one row.

Use a workflow tool instead when the capability needs multiple operations, transformations, branches, loops, durable state, triggers, a custom typed contract, or reuse outside the agent. Do not build a one-node workflow when the same ready action already fits.

## Discover exact identities

```bash
cai integration search "<app or capability>" --json
cai integration actions "<operation>" --integration <integrationSlug> --json
cai integration action --integration <integrationSlug> --key <actionKey> --json
```

Preserve each returned `nodeSlug` and `key` pair exactly, then inspect that action's static props before using it. Never derive either identifier from a display label. Narrow the query or raise `--limit` when more matches exist.

## Select a connection

```bash
cai connection list --integration <integrationSlug> --json
```

Use an active connection whose `integration` exactly matches the integration slug. If none exists:

```bash
cai connection connect <integrationSlug>
```

MCP: call `connection_connect_url`, show its `url` as a markdown link, ask the user to finish, then confirm with `connection_list` using its `integration`; it has no `open` or `wait` parameter.

The connection handoff keeps provider credentials out of the agent context. The CLI may open the URL; MCP returns it. Re-list connections after the user finishes, then attach the action.

Rename or change the default with `cai connection update <id> [--name "<displayName>"] [--default|--no-default]` (MCP: `connection_update`); delete only intentionally with `cai connection delete <id> --yes` (MCP: `connection_delete`).
Deleting a connection immediately removes authorization from bound nodes and agent tools.

A missing connection is never a reason to substitute `registry_http-http-request` for a catalog component. For a workflow node, add the real component and leave it unconnected; `missingConnection: true` is a valid handoff state. For a direct action, preserve the selected integration and action and stop at the connection handoff. Give the user the returned connection URL; on CLI, `cai connection connect <slug>` obtains it. A raw endpoint mentioned by the user is background context, not an instruction to hand-roll the call.

## Decide approval before attaching

Attached actions run without a human approval step unless `requireConfirmation` is `true`. When attaching an action that sends, posts, spends, or deletes, choose `true` or explicitly tell the user that it will run unattended. Mention the confirmation choice in the report to the user.

- Prefer confirmation for permission changes, irreversible writes, or an ambiguous destination.
- Usually omit confirmation for read-only retrieval and low-risk, reversible operations the user clearly asked the agent to perform autonomously.

```bash
cai agent add-action <agentId> \
  --integration <integrationSlug> \
  --action <actionKey> \
  --connection <connectionId> \
  --description "Use when ..." \
  --require-confirmation \
  --json
```

MCP: call `agent_tool_action_add`; `requireConfirmation` is optional and defaults to `false`. On CLI, `--require-confirmation` selects `true` and omission preserves the `false` default. Do not attach the same action twice.

Read the saved grant back:

```bash
cai agent tools <agentId> --json
```

Verify `toolType`, integration, action key, connection, schema tier, description, and confirmation policy. The hosted runtime configures dynamic action properties when it invokes the action; the builder does not preconfigure runtime-only action arguments here.

To rebind the live authorization grant or change a versioned policy:

```bash
cai agent update-tool <agentId> --tool <toolId> --connection <connectionId> --json
cai agent update-tool <agentId> --tool <toolId> --require-confirmation --json
cai agent update-tool <agentId> --tool <toolId> --no-confirmation --json
```

MCP: call `agent_tool_update`; use `connectionId` and `requireConfirmation: true` or `requireConfirmation: false`.

`--connection` can redirect the published action immediately because authorization is live-bound. Description and confirmation-policy edits remain draft changes until the agent is published.

## Verify through the hosted agent

Use a representative draft conversation:

```bash
cai agent test <agentId> --draft --message "<request>" --json
cai agent conversation history <conversationId> --json
```

MCP: call `agent_test` with `rail: draft` and `waitSeconds`.

A direct action runs inline and does not create a workflow execution ID. Evidence is the persisted conversation, its direct-action tool/run events, the returned result, and any observed external side effect. Never invent a workflow execution ID for it.

If confirmation is required, inspect the pending action and exact arguments before approving or denying its request. A successful assistant reply alone does not prove the intended action ran.

## Publishing and sharing semantics

Publishing snapshots the action identity, description, and approval policy into the agent version. The connection and authorization grant remain live security bindings: rebinding can redirect the published action, while breakage, revocation, or detachment can stop it immediately without another agent publish. Detachment also removes the draft tool; publish afterward to remove the unavailable tool from the published list.

Organization members use the published agent, never its mutable draft. Direct actions run through the agent owner's connected external account. Controller AI usage draws from the organization's shared usage pool, regardless of which member calls the agent. State that credential boundary before organization sharing.

## Before you finish

```bash
cai agent tools <agentId> --json
cai agent conversation history <conversationId> --json
```

- Tools must show one intended direct attachment with the exact returned identities, connection, description, and confirmation policy.
- History must contain the persisted action event, exact arguments, and result; confirm any requested external effect separately.
- No wrapper workflow should exist unless the capability actually required orchestration or reuse.
