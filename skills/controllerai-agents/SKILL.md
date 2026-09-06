---
name: controllerai-agents
description: Design, assemble, test, publish, and share user-facing Controller AI agents with ready integration actions, workflow tools, instructions, and knowledge files. Use when creating or editing an agent, choosing its tool architecture, attaching or updating tools, verifying draft/live conversations, or publishing and sharing an agent safely.
---

# Controller AI agents

An agent has one mutable draft and immutable published versions. Build instructions, tools, permissions, and files on the draft; verify in Test; publish last. Draft edits never update Live by themselves.

The `cai ...` examples are canonical grammar. Use the transport selected in `controllerai-start` and its MCP tool map; never mix transports in one task or call Controller AI REST routes directly.

## Rules that break tasks when skipped

- Attach one ready action directly instead of wrapping it; see Choose the capability architecture.
- Choose confirmation before attaching a consequential tool; see Create and assemble the draft.
- Use one conversation on an explicit rail for each verification task; see Verify the draft as an agent.
- Resolve pending requests in that same conversation; see Verify the draft as an agent.
- Enablement is runtime state, not publication; see Create and assemble the draft.
- Publish authorized workflow dependencies, preflight, then publish the agent; see Publish in dependency order.
- Draft-only means stop before publish; draft evidence never proves Live. See Publish in dependency order.

## 1. Choose the capability architecture

- Use `controllerai-integration-actions` when one existing app operation completes it.
- Use a workflow tool for multiple steps, transformations, branching, looping, durable state, triggers, a custom typed contract, or reusable business logic.
- Put judgment, tool-selection rules, tone, and escalation in instructions; put stable sources in knowledge files.

Do not create a one-node workflow merely to expose a ready action, or replace orchestration with several independent direct actions when ordering or data handling must be deterministic.

## 2. Build and verify capabilities

For a ready action, follow `controllerai-integration-actions`: discover exact identities, select a connection, decide confirmation, then attach.

For every workflow tool, load the relevant workflow skills and test the development state before attaching:

```bash
cai use <workflowId>
cai flow get <flowId> --json
cai run flow <flowId> --inputs values.json --json
```

MCP: there is no pin; pass `workflowId` to `flow_get` and `run_flow`, pass parsed `inputs`, and set `waitSeconds`.

Capture the successful execution ID. Inputs become tool parameters and outputs its result. Require a nonempty output contract and verify the run returns it.

## 3. Create and assemble the draft

Write short instructions for role, tool use, inputs/results, safety, response style, and errors. Require truthful failure reporting and at most one retry.

Read [references/instructions-and-versioning.md](references/instructions-and-versioning.md) before publishing or explaining Test versus Live behavior.

```bash
cai agent create --name "<agent name>" --instructions-file instructions.md --json
```

MCP: `agent_create` takes complete Markdown in `instructions`; it cannot read `instructions.md`.

```bash
cai agent add-action <agentId> \
  --integration <integrationSlug> \
  --action <actionKey> \
  --connection <connectionId> \
  --description "Use when ..." \
  --json

cai agent add-workflow <agentId> \
  --workflow-id <workflowId> \
  --flow <flowId> \
  --description "Use when ...; accepts ...; returns ..." \
  --json
```

MCP: use `agent_tool_action_add` or `agent_tool_workflow_add`; `requireConfirmation` is optional and defaults to `false`.

```bash
cai agent add-file <agentId> --file <path> --space <knowledge|shared> --json
cai agent files <agentId> --space <knowledge|shared> [--live] --json
cai agent tools <agentId> --json
cai agent get <agentId> --json
```

MCP: use `agent_knowledge_file_add` or `agent_workspace_file_add` with inline `content`; `agent_file_list` uses `environment: dev|live`. Shared runtime files update live immediately.

The default is no approval: attached actions and workflow tools run without a human approval step unless `requireConfirmation` is `true`, and omitting `--require-confirmation` has the same behavior. When attaching an action that sends, posts, spends, or deletes, choose `true` or explicitly tell the user that it will run unattended. Mention the confirmation choice in the report to the user; `cai agent update-tool` can change it later.

Keep the two modes straight:

| Context | Test / draft conversation | Live / shared conversation |
| --- | --- | --- |
| Agent configuration | Current mutable draft | Latest published immutable agent version |
| Workflow tools | Current workflow development heads | Each workflow's latest live release; never pinned |
| Reference files | Current draft knowledge files | Knowledge frozen into the published agent version |
| Runtime files | Current live runtime files, overlaid by development/test runtime files at the same path | Current live runtime files only |
| Access | Owner only | Owner, plus organization members when shared; non-owners never see the draft |

Version pinning for workflow tools does not exist. Live tools track the latest release. If asked to freeze one, explain this, attach with policy `latest`, and never abandon the attachment.

Publishing snapshots versioned fields, moves the live pointer, and restamps every existing live conversation. It does not snapshot the runtime model. This cutover interrupts in-flight answers and expires pending approvals; time it accordingly.

Runtime lifecycle changes do not require publishing:

```bash
cai agent update <agentId> --disabled --json
cai agent update <agentId> --enabled --json
cai agent delete <agentId> --confirm --json
```

MCP: draft edits use `agent_update`; runtime uses `agent_set_enabled`; `agent_delete` has no `confirm`. Disable pauses runtime. Soft delete revokes access and requests session termination; there is no CLI restore.

## Runtime usage boundary

The user's coding-agent subscription covers builder reasoning, but hosted agent tests and conversations consume Controller AI runtime usage from the organization's pool, and connected providers may add charges or side effects. Say so before starting a hosted test.

## 4. Verify the draft as an agent

Saving the draft updates the agent's Test environment — there is no separate deploy step. Test conversations run each workflow tool's DEVELOPMENT version.

```bash
cai agent test <agentId> --draft --message "<representative request>" --json
cai agent conversation history <conversationId> --json
```

MCP: `agent_test` requires `rail: draft` or `rail: live` and takes `waitSeconds`.

`agent test` always creates a conversation. Use it for a one-shot check. When a task may need follow-up or approval, create one conversation on the requested rail, capture its ID, and reuse it:

```bash
cai agent conversation create <agentId> --draft --title "<test case>" --json
cai agent conversation send <conversationId> --message "<request>" --wait --json
```

Do not resend the request or create a replacement after an approval pause. A required post-publish Live check is a separate rail task and gets one fresh Live conversation.

Test positive and negative tool selection, exact arguments, results, errors, and knowledge. Offer the returned `appUrl`; on MCP, call `agent_present` and relay the returned card.

- For a workflow tool, correlate a side-effectful call with its terminal workflow execution ID.
- For a direct integration action, inspect persisted conversation action events/run result and the intended external effect. Direct actions do not create workflow execution IDs.

For a pending approval, inspect the exact request first:

```bash
cai agent conversation approve <conversationId> --request <requestId> --json
cai agent conversation deny <conversationId> --request <requestId> --message "<reason>" --json
```

The server resolves the tool from the request ID. Do not approve merely to continue. Resolve every pending request before publishing; approval or denial resumes this conversation, while publishing expires it.

After `cai agent remove-tool`, test one request that must no longer use the removed capability and one retained capability.

## 5. Publish in dependency order

For attached workflows only:

```bash
cai use <workflowId>
cai workflow status --json
cai workflow publish --name "<release>" --json
```

MCP: there is no `use`; pass `workflowId` to `workflow_status` and `workflow_publish`.

Publish a workflow only when its verified development state must go live and the user authorized it. A workflow publish already followed by a live agent changes that tool's live behavior immediately; do not republish the agent merely to pick it up.

For a draft-only request, stop after Test evidence. Do not publish a workflow or agent merely because it is ready.

Then publish the agent:

```bash
cai agent publish-preflight <agentId> --json
cai agent publish <agentId> --name "<release>" --json
cai agent versions <agentId> --json
cai agent test <agentId> --live --message "<representative request>" --json
```

MCP: `agent_test` requires `rail: draft` or `rail: live` and takes `waitSeconds`.

`publish-preflight` is the arbiter of structural readiness. Report blockers and stop. With none and an authorized release, publish, except an error you actually watched happen; that blocks even a green preflight.

Email verification is not a publishing prerequisite. An unclaimed agent-created account may publish; email claiming only raises its monthly usage cap and enables account recovery.

Use one loop: edit draft, read back, verify Test, publish, verify Live. Published versions have no rollback. Recover forward from `agent versions`; restore workflow definitions with history/restore, then republish. Workflow releases, action grants, and runtime files are live-bound; see [references/instructions-and-versioning.md](references/instructions-and-versioning.md). After detaching an action, publish to remove it from the published list.

Draft evidence does not prove the live agent. Test is owner-only; shared users always run the latest published agent version and never see draft instructions, tools, or knowledge.

## 6. Offer Slack and organization sharing

After publish, offer product setup for Slack. Agents remain private unless the user requests organization access:

```bash
cai agent visibility <agentId> --value organization --json
```

Members use the published version. Direct actions use the owner's external account and organization usage pool. Grant changes affect Live immediately; private visibility revokes access.

## Before you finish

```bash
cai agent tools <agentId> --json
cai agent conversation list <agentId> --mode dev --json
cai data query --type custom.<type> --env dev --json
cai agent conversation history <conversationId> --json
cai agent publish-preflight <agentId> --json
cai agent versions <agentId> --json
cai agent test <agentId> --live --message "<representative request>" --json
```

Run the last three commands only for an authorized release. For Live verification, use `--mode live` and `--env live` in the same pair.

- Conversation list must show the captured ID on the requested rail, without retry-created duplicates.
- History must show the intended tool arguments, terminal result, and no unresolved permission request.
- For stateful behavior, the same-rail data query must show the expected record; an assistant reply is not proof.
- A release requires clean preflight, agent version read-back, and one fresh Live conversation. Draft-only work must remain unpublished.

Report the agent ID/version, tools and confirmation policies, execution or direct-action evidence, rail state, connections, access, and anything unverified.
