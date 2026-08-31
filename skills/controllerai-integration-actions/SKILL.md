---
name: controllerai-integration-actions
description: Discover, connect, attach, update, and verify ready integration actions used directly by Controller AI agents. Use when an agent needs one existing app operation without a workflow, when choosing between a direct action and a workflow tool, or when changing an action's connection or per-run approval policy.
---

# Controller AI integration actions

In MCP hosts, send each exact literal `cai ...` command through the single `exec` tool; never translate it into another grammar.

Ready integration actions give a hosted agent one app operation without first building a workflow. Use only the `cai` commands below; do not call Controller AI REST or MCP routes directly.

## Choose direct action or workflow

Use a direct action for one existing operation such as sending one message, creating one record, or adding one row.

Use a workflow tool instead when the capability needs multiple operations, transformations, branches, loops, durable state, triggers, a custom typed contract, or reuse outside the agent. Do not build a one-node workflow when the same ready action already fits.

## Discover exact identities

```bash
cai integration search "<app or capability>" --json
cai integration actions <integrationSlug> --search "<operation>" --json
```

Preserve the returned integration `slug` and action `key` exactly. Never derive either from a display label. Continue with `--after <nextCursor>` when pagination reports more results.

## Select a connection

```bash
cai connection list --integration <integrationSlug> --json
```

Use an active connection whose `nodeSlug` exactly matches the integration. If none exists:

```bash
cai connection connect <integrationSlug>
```

The browser handoff keeps provider credentials out of the agent context. Re-list connections after the user finishes, then attach the action.

A missing connection is never a reason to substitute `registry_http-http-request` for a catalog component. For a workflow node, add the real component and leave it unconnected; `missingConnection: true` is a valid handoff state. For a direct action, preserve the selected integration and action and stop at the connection handoff. Give the user `cai connection connect <slug>`. A raw endpoint mentioned by the user is background context, not an instruction to hand-roll the call.

## Decide approval before attaching

The product default is to run without per-call approval. Do not accept that default mechanically. Inspect the action name, description, pricing, and intended use, then decide:

- Require confirmation for deletion, purchases or spend, external communications, permission changes, irreversible writes, or an ambiguous destination.
- Usually omit confirmation for read-only retrieval and low-risk, reversible operations the user clearly asked the agent to perform autonomously.

Tell the user which policy was selected and why when it has a meaningful consequence.

```bash
cai agent add-action <agentId> \
  --integration <integrationSlug> \
  --action <actionKey> \
  --connection <connectionId> \
  --description "Use when ..." \
  --require-confirmation \
  --json
```

Omit `--require-confirmation` only after making the decision; the CLI still sends an explicit `false`. Do not attach the same action twice.

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

`--connection` can redirect the published action immediately because authorization is live-bound. Description and confirmation-policy edits remain draft changes until the agent is published.

## Verify through the hosted agent

Use a representative draft conversation:

```bash
cai agent test <agentId> --draft --message "<request>" --json
cai agent conversation history <conversationId> --json
```

A direct action runs inline and does not create a workflow execution ID. Evidence is the persisted conversation, its direct-action tool/run events, the returned result, and any observed external side effect. Never invent a workflow execution ID for it.

If confirmation is required, inspect the pending action and exact arguments before approving or denying its request. A successful assistant reply alone does not prove the intended action ran.

## Publishing and sharing semantics

Publishing snapshots the action identity, description, and approval policy into the agent version. The connection and authorization grant remain live security bindings: rebinding can redirect the published action, while breakage, revocation, or detachment can stop it immediately without another agent publish. Detachment also removes the draft tool; publish afterward to remove the unavailable tool from the published list.

Organization members use the published agent, never its mutable draft. Direct actions run through the agent owner's connected external account. Controller AI usage draws from the organization's shared usage pool, regardless of which member calls the agent. State that credential boundary before organization sharing.
