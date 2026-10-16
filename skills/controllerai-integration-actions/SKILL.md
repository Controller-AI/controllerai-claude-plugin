---
name: controllerai-integration-actions
description: Discover, connect, attach, update and verify ready integration actions that a Controller AI agent calls directly. Use when an agent needs one existing app operation without a workflow, when choosing between a direct action and a workflow tool, or when changing an action's connection or per-run approval policy.
---
<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->

# Integration actions

## Use this when

- An agent needs one existing app operation: send one message, create one record, add one row, look something up.
- You must decide between attaching a ready action and building a workflow tool.
- An attached action needs another connection, a different approval policy, or proof that it runs.

Building the agent itself belongs to `controllerai-agents`; a workflow node for the same app belongs to `controllerai-node-config`.

## How it works

**A ready action is one operation.** The catalog exposes provider integration actions and native registry actions that an agent can call directly, without a workflow; a native action is attachable when the catalog marks it `agentAttachable`. Most native actions need no connection: they run on the platform's paid rails, bill the user's usage balance per call, and describe the charge in `usagePricingDescription`. A native action that does need authorization uses the platform connection flow (`connection_connect_url`), not Pipedream. Use a workflow tool instead when the capability needs several operations, transformation, branching, loops, durable state, a trigger, a custom typed contract, or reuse outside the agent. Never wrap a ready action whose contract already fits in a one-node workflow, and never replace deterministic orchestration with several independent direct actions.

**Identity is a pair.** An action is the exact integration slug plus its action key, as returned by discovery; a display label is not an identity, and a similar integration with a familiar name is not evidence of the capability. When the catalog says the action needs a connection, it binds to one whose `integration` matches that slug exactly.

**Runtime.** The agent runtime configures the action's dynamic arguments when it invokes the action; nothing is preconfigured at attachment beyond identity, any required connection, description and approval policy. A direct action runs inline in the conversation and creates no workflow execution: the evidence is the persisted conversation, its action events with arguments and result, and the external effect.

**Approval policy.** An attached action runs without a human approval step unless `requireConfirmation` is `true`; the default is no approval. When confirmation is on, the runtime pauses with a pending request that the user approves or denies with the exact arguments visible.

**Slack approvals.** Send approval requests to Slack through `slackRoute`. Routes publish with the agent; Test conversations use the draft, live conversations the published version. The route lists who can answer in `slackRoute.approvers`, otherwise anyone in the channel can answer. Chat-in-Slack approvers also count; the app's approval card still works.

**Live bindings.** Publishing snapshots the action identity, description and approval policy into the agent version. When an action needs a connection, that connection and its authorization grant are live security bindings: rebinding redirects the published action immediately, and revocation, breakage or detachment stops it immediately without an agent publish. Detachment has the same immediate effect on a connectionless action, always removes the draft tool, and a later publish removes it from the published list. Confirmation is strictest-wins: tightening the grant to require confirmation blocks the published action at once until a publish carries the new policy, while relaxing it waits for the publish. Organization members use the published agent, never the draft; connected actions run through the owner's connected account and the organization's shared usage pool, while connectionless native actions use the platform's paid rails and bill that pool whoever invokes them.

## Rules

1. **Capture and reuse the exact identities**: integration slug, action key, connection ID when required, tool ID. Narrow the query or raise the result limit when more matches exist; never derive an identifier from a label.
2. **A required missing connection is a handoff, not a substitution.** Keep the chosen integration and action, give the platform authorization route, stop there, and never request provider credentials in chat. Never swap in `registry_http-http-request`, which is never agent-attachable; a raw endpoint the user mentions is context, not an instruction to hand-roll the call.
3. **Decide the approval policy before attaching.** For an action that sends, posts, spends or deletes, choose `true` or explicitly tell the user it will run unattended; prefer confirmation for permission changes, irreversible writes or an ambiguous destination, and omit it for reads and low-risk reversible operations the user asked to run autonomously. Mention the confirmation choice in the report to the user.
4. **Attach once.** Read the tool list before and after; the final list holds one intended direct attachment.
5. **A stored active connection is not proof of authorization.** For an action that needs one, only a representative action test proves the provider still accepts it.
6. **Verify what the request asked for, where it lives.** Discovery proves metadata; a blocked attachment proves the honest handoff and the independent work; an attachment proves the saved grant; only an authorized runtime test proves the external effect, and for that the action event, its arguments and result, and the external state are the evidence, never a reply that says it ran. Inspect the external state before repeating a failed or uncertain mutation.
7. **Say whose account and balance run the action** before organization sharing: connected actions use the owner's account; connectionless native actions use the platform's paid rails; calls use the organization's shared usage balance regardless of caller.

## Recipes

### Discover

- `integration_search` — `query`, `agentAttachable`, `limit`, `kind`
- `integration_action_list` — `query`, `agentAttachable`, `integration`, `limit`
- `integration_action_get` — `integration`, `key`

`integration_search` ranks integrations (`kind` narrows to actions or events); `integration_action_list` ranks actions across one or all integrations; `integration_action_get` returns the static metadata and props of one exact action. Read `agentAttachable`, the exact props and, for native actions, whether a connection is needed and `usagePricingDescription` before choosing.

### Select or request a connection

- `connection_list` — `integration`

Skip this step when the catalog says the native action needs no connection. Offer a `shared: true` row (a teammate's account) only when the user has none; use it after a yes.

Use the active connection whose `integration` matches exactly. If none exists, `connection_connect_url` (it takes the `nodeSlug`) returns a `url`: show it as a markdown link, ask the user to finish, then `connection_list` with `integration` and use the new ID.

### Attach

- `agent_tool_action_add` — `agentId`, `action`, `connectionId`, `description`, `integration`, `requireConfirmation`, `slackRoute`
- `agent_tool_list` — `agentId`

`agent_tool_action_add` takes `requireConfirmation`, optional and `false` by default; pass `connectionId` exactly when the action needs a connection.
The description tells the agent when to choose the tool. Read the saved grant back: `toolType`, integration, action key, any required connection, description and confirmation policy.

### Update

- `agent_tool_update` — `agentId`, `connectionId`, `description`, `requireConfirmation`, `toolId`, `slackRoute`

`agent_tool_update` takes `toolId` and at least one of `connectionId`, `description` or `requireConfirmation: true|false`; an omitted field keeps its value, so a connection-only update never changes the approval policy.
A connection change redirects a connected published action at once. A description change waits for the agent publish. A policy change is strictest-wins: turning confirmation on blocks the published action immediately until the next publish carries it, turning it off takes effect only at that publish.

### Verify in a draft conversation

- `agent_test` — `agentId`, `message`, `model`, `waitSeconds`, `title`, `rail`
- `agent_conversation_history` — `conversationId`, `last`
- `agent_conversation_approve` — `conversationId`, `requestId`
- `agent_conversation_deny` — `conversationId`, `message`, `requestId`

`agent_test` takes `rail: "draft"` and `waitSeconds`; approve and deny take the `requestId` from the pending request.
`agent_test` is a one-shot that opens a new conversation; when a follow-up or an approval is expected, create one draft conversation and send into it as in `controllerai-agents`. When confirmation is on, inspect the pending action and its exact arguments, then approve or deny in that same conversation and read the history for the terminal result. Direct actions do not create workflow execution IDs: never invent one.

## Verify

For an attachment, the tool list shows exactly one intended direct attachment with the exact identities, any required connection, description and confirmation policy; for an authorized runtime test, the draft conversation history holds the action event, its arguments and its result, and the external effect was checked separately; for discovery alone, the exact metadata was read and nothing was changed. No wrapper workflow exists unless orchestration or reuse required it.

## Report

Which action is attached to which agent, any connection and the confirmation choice, what the draft test proved, whose account or paid rail and usage balance the action will use, and what remains: a connection to authorize, a publish, or a live verification.
