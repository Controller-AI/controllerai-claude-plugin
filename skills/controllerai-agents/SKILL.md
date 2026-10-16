---
name: controllerai-agents
description: Design, assemble, test, publish and share user-facing Controller AI agents built from instructions, knowledge files, workflow tools and ready integration actions. Use when creating or editing an agent, choosing its tool architecture, attaching tools, verifying draft or live conversations, or publishing and sharing it.
---
<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->

# Agents

## Use this when

- The user wants something people chat with: an assistant, a copilot, a file-enrichment helper, an internal tool with judgment.
- An existing agent needs new instructions, tools or knowledge, or must be verified, published or shared.
- A capability must be placed: in the agent's instructions, in a workflow tool, in a direct action, or in a knowledge file.

Direct actions belong to `controllerai-integration-actions`; the workflows behind workflow tools to `controllerai-workflows`; the wider deliverable choice to `controllerai-start`.

## How it works

**What an agent is.** A person chats with it; it reasons, reads conversation attachments and reference knowledge, calls tools, and creates files for the user. It is the right shape for judgment, conversation and file handling; deterministic business rules, custom workflow data and multi-step processing belong in workflow tools it calls, and one catalog-marked operation from a provider integration or native registry service is a direct action. Both direct-action sources follow the same attachment and approval rules; a native action may not need a connection.

**One draft, immutable versions.** Editing changes only the mutable draft, and saving the draft updates the owner-only Test environment at once. Publishing snapshots the display name, instructions, tool identities, descriptions, approval policies and knowledge files into a new immutable version, moves the live pointer, and restamps every existing live conversation to it; that cutover interrupts in-flight answers and expires pending approvals. The runtime model is not part of the snapshot. There is no rollback: a bad version is corrected forward.

**Test versus Live.**

| Context | Test (draft) conversation | Live (shared) conversation |
| --- | --- | --- |
| Agent configuration | Current mutable draft | Latest published immutable agent version |
| Workflow tools | Each workflow's current development head | Each workflow's latest live release; never pinned |
| Reference files | Current draft knowledge files | Knowledge frozen into the published agent version |
| Runtime files | Current live runtime files, overlaid by development/test runtime files at the same path | Current live runtime files only |
| Access | Owner only | Owner, plus organization members when shared; non-owners never see the draft |

**Three kinds of files.** Conversation attachments belong to one chat. Knowledge is versioned reference material in the draft: text types only (`txt`, `md`, `json`, `csv`, `html`, `xml`), each added once under the name the agent should see. When the user asks for a PDF or another unsupported format, store a supported text type under the plain requested name and say why; a name that still carries the wrong extension is not the same document. Shared runtime files are live workspace state, visible without a publish, and no substitute for an upload. Paths come from returned metadata; runtime mounts can prefix them.

**Live-bound state.** Enabling or disabling is runtime state, independent of publication: disabling pauses live use, while the owner's Test conversations still work. A workflow publish changes every live agent that follows that workflow without an agent publish, and live workflow resolution is cached per session, so an in-flight turn may finish on the prior release. When a direct action needs a connection, that connection is a live grant: rebinding redirects it and revocation blocks it immediately. Detachment blocks every direct action immediately and still needs a later agent publish to remove the tool from the published list.

**Slack approvals.** Send approval requests to Slack through `slackRoute`. Routes publish with the agent; Test conversations use the draft, live conversations the published version. The route lists who can answer in `slackRoute.approvers`, otherwise anyone in the channel can answer. Chat-in-Slack approvers also count; the app's approval card still works.

**Runtime usage.** Test and live conversations consume the organization's runtime usage and cause the connected providers' side effects; the builder's own subscription pays only for building.

## Rules

1. **Instructions hold judgment; tools hold work; knowledge holds sources.** Each tool description says when to use it, what to pass, what comes back.
2. **Test every workflow tool on development before attaching it**, with a non-empty output contract the run actually returns, unless the user declined that test and asked for the attachment anyway.
3. **Decide the approval policy before attaching a consequential tool.** The default is no approval, which is right for reads and searches; for a tool that sends, posts, spends or deletes, choose `requireConfirmation` or explicitly tell the user it will run unattended, and mention the choice in the report to the user. The policy is a runtime setting for the agent's users, not a permission gate on your own work, so honor a policy the user already specified.
4. **Never promise a pinned workflow tool.** Live tools follow the latest release and nothing else is supported; when asked to freeze one, explain that, ask whether following the latest release is acceptable, and finish the independent work meanwhile.
5. **One conversation per verification task, on an explicit rail.** Reuse its ID for follow-ups and approvals; never resend the request or open a replacement after an approval pause, and never approve merely to continue.
6. **Resolve the task's pending requests, in its conversation and within the user's scope, before publishing**, because a publish expires the unexecuted requests in the affected live conversations.
7. **Draft-only means stop at Test evidence.** Readiness authorizes nothing; a draft never proves the live agent.
8. **Publish in dependency order**: attached workflows the user authorized, then agent preflight, then the agent, then one fresh live conversation. A preflight blocker on an unfinished workflow tool is cleared by detaching that tool; never delete its flow or publish the workflow to make preflight pass, because both release or destroy work the user did not ask to ship.
9. **Failure instructions are specific.** Say what failed and what was attempted, retry at most once, distinguish reads from rejected validation from uncertain mutations, and name the integration when a connection breaks.
10. **Keep internals out of the runtime user's view**: no workflow, flow, node, rail, prop or version names in instructions or replies.

## Recipes

### Create the draft

- `agent_list` — `ownership`, `search`
- `agent_create` — `disabled`, `instructions`, `name`
- `agent_get` — `agentId`

`agent_create` takes the complete Markdown in `instructions`; there is no file to read.
A draft is private and by default enabled on creation, so the user can open it and start testing at once; nothing is live until a publish. Write instructions from the template in [references/instructions-and-versioning.md](references/instructions-and-versioning.md) and delete every sentence that does not change behavior.

### Attach tools and knowledge

- `run_flow` — `flowId`, `confirmLive`, `inputs`, `target`, `waitSeconds`, `workflowId`
- `agent_tool_workflow_add` — `agentId`, `description`, `flow`, `requireConfirmation`, `workflowId`, `slackRoute`
- `agent_knowledge_file_add` — `agentId`, `content`, `name`, `replace`, `type` / `agent_workspace_file_add` — `agentId`, `content`, `name`, `replace`, `space`, `type`
- `agent_tool_list` — `agentId`
- `agent_file_list` — `agentId`, `cursor`, `environment`, `folderPrefix`, `limit`, `live`, `search`, `space`

- `agent_file_remove` — `agentId`, `environment`, `fileId`, `path`, `space`

`agent_tool_workflow_add` and `agent_tool_action_add` take `requireConfirmation`, optional and `false` by default. Files travel inline: `agent_knowledge_file_add` (`name`, `type`, `content`) for knowledge, `agent_workspace_file_add` with `space` for shared runtime files (live immediately). A name that is already stored becomes a numbered copy (`name-2.md`) unless `replace` is `true`, so read the returned `file.sandboxPath` before a second call. `agent_file_remove` deletes one file by the exact `path` (or `fileId`) that `agent_file_list` returns and refuses an ambiguous or unknown path; `agent_file_list` filters runtime files by `environment`.
Flow inputs become the tool's parameters and its outputs the result.

### Verify in Test

- `agent_test` — `agentId`, `message`, `model`, `waitSeconds`, `title`, `rail`
- `agent_conversation_create` — `agentId`, `model`, `title`, `rail`
- `agent_conversation_send` — `conversationId`, `message`, `waitSeconds`
- `agent_conversation_history` — `conversationId`, `last`
- `agent_conversation_approve` — `conversationId`, `requestId`
- `agent_conversation_deny` — `conversationId`, `message`, `requestId`

`agent_test` takes `rail: "draft"` or `"live"` and `waitSeconds`; `agent_conversation_send` takes `waitSeconds`; when a wait returns unfinished, poll `agent_conversation_status` and read `agent_conversation_history` for that conversation instead of sending again; approve and deny take the `requestId` of the pending request.
`agent_test` is a one-shot that always creates a conversation; use `agent_conversation_create` and `agent_conversation_send` when follow-up or an approval may be needed.
 Test positive and negative tool selection, exact arguments, results, errors and knowledge. A workflow tool's side effect is proved by its terminal workflow execution; a direct action by its persisted action events, result and external effect, never by an execution ID it does not have. After removing a tool, test one request that must no longer use it and one that still uses a retained tool. Say before the first test that it consumes runtime usage and may reach providers.

### Publish and share

- `workflow_status` — `workflowId`
- `workflow_publish` — `acknowledgeDualDispatch`, `acknowledgeValidationErrors`, `name`, `workflowId`
- `agent_get` — `agentId`
- `agent_publish_preflight` — `agentId`
- `agent_publish` — `agentId`, `name`
- `agent_version_list` — `agentId`
- `agent_test` — `agentId`, `message`, `model`, `waitSeconds`, `title`, `rail`
- `agent_visibility_set` — `agentId`, `value`

`workflow_status` and `workflow_publish` take the explicit `workflowId` of each attached workflow; `agent_get` reports `hasUnpublishedChanges`; `agent_version_list` is the release history; `agent_visibility_set` takes `value`.
Publish an attached workflow only when its verified development state must go live and the user authorized it. `hasUnpublishedChanges` on the agent read says whether a publish would change anything; do not publish an unchanged draft. Preflight is the arbiter of structural readiness: report blockers and stop; with none and an authorized release, publish unless an error you watched happen remains. Then verify with a fresh live conversation. Agents stay private unless the user asks for organization access: members use the published version through the owner's connected accounts and the organization's usage pool, and private visibility revokes access immediately. After a publish, offer the product's Slack setup.

### Runtime state

- `agent_update` — `agentId`, `instructions`, `name` / `agent_set_enabled` — `agentId`, `enabled`
- `agent_delete` — `agentId`

`agent_set_enabled` takes `enabled: false` to pause or `enabled: true` to resume; `agent_update` edits the draft; `agent_delete` has no confirmation parameter.
Disable pauses live use without a publish; the owner's Test conversations keep working. Delete is a soft delete that revokes access and asks active sessions to stop; there is no restore.

### Hand over

Call `agent_present` for the finished agent and relay the returned card with its name, purpose, tools and link.

## Verify

The tool list shows the intended tools with exact identities, descriptions and confirmation policies; the knowledge list shows the intended files; the agent read-back matches the instructions; the captured conversation exists on the requested rail without duplicates, its history shows the intended arguments, terminal results and no unresolved request, and stateful behavior is proved by a same-rail record query; a release shows a clean preflight, the new version in the version list, and one fresh live conversation.

## Report

The agent ID and version, its tools with their confirmation policies, what the Test and live conversations proved, connections and access, and what is still unverified or unpublished.
