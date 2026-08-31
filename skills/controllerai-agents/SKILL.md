---
name: controllerai-agents
description: Design, assemble, test, publish, and share user-facing Controller AI agents with ready integration actions, workflow tools, instructions, and knowledge files. Use when creating or editing an agent, choosing its tool architecture, attaching or updating tools, verifying draft/live conversations, or publishing and sharing an agent safely.
---

# Controller AI agents

An agent has one mutable draft and immutable published versions. Capabilities are ready integration actions, workflow tools, or both. Build on the draft, verify through a draft conversation, publish last. Editing the draft never updates the live agent by itself.

Agent design is a lifecycle — hold the whole arc:

1. Write the instructions (the agent's system prompt).
2. Add tools — workflow flows and/or ready integration actions.
3. Set each tool's permission level WITH the user — HITL is a design decision.
4. Set tool descriptions (powerful for workflow tools).
5. Add reference files (`txt, md, json, csv, html, xml`).
6. Save — the draft IS the test environment.
7. Test it — and let the user try it in the product.
8. Publish when it looks good.
9. Offer Slack access and organization sharing.

Use exact `cai ...` commands only. In MCP hosts, send them through `exec`; never call Controller AI REST or MCP routes directly.

## 1. Choose the capability architecture

For each capability:

- Use `controllerai-integration-actions` when one existing app operation completes it.
- Use a workflow tool for multiple steps, transformations, branching, looping, durable state, triggers, a custom typed contract, or reusable business logic.
- Put conversational judgment, when-to-use rules, tone, and escalation in agent instructions.
- Put stable source material in a knowledge file.

Do not create a one-node workflow merely to expose a ready action, or replace orchestration with several independent direct actions when ordering or data handling must be deterministic.

## 2. Build and verify capabilities

For a ready action, follow `controllerai-integration-actions`: discover the exact integration/action, select a connection, decide confirmation policy, then attach it.

For every workflow tool, load the relevant workflow skills and test the development state before attaching:

```bash
cai use <workflowId>
cai flow get <flowId> --json
cai run flow <flowId> --inputs values.json --json
```

Capture the successful workflow execution ID. Inputs become tool parameters; outputs become its result.
Before attaching, require a nonempty output contract and verify the completed run returns it; otherwise the agent receives an empty result.

## 3. Create and assemble the draft

Write short instructions covering role, tool-selection rules, expected inputs/results, safety boundaries, response style, and error behavior. Require: "If an action returns an error, tell the user what failed and what you were trying to do; do not retry more than once."

Read [references/instructions-and-versioning.md](references/instructions-and-versioning.md) before publishing or explaining Test versus Live behavior.

```bash
cai agent create --name "<agent name>" --instructions-file instructions.md --json

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

cai agent add-file <agentId> --file <path> --space <knowledge|shared> --json
cai agent files <agentId> --space <knowledge|shared> [--live] --json
cai agent tools <agentId> --json
cai agent get <agentId> --json
```

Adding with `--space shared` updates the live runtime file space immediately without publishing; an active agent sees it on the next turn, a disabled agent after enablement.

Permission is part of agent design. Settle each tool's approval level WITH the user before attaching: present the tool list and ask which steps need a human in the loop ("should sending the email require your approval each time, or run automatically?"). The default is no approval. Add `--require-confirmation` for consequential writes, deletion, external communications, permission changes, purchases/spend, or an ambiguous destination; `cai agent update-tool` changes it later. Omitting the flag must be an informed autonomy decision, not an unnoticed default.

Keep the two modes straight:

| Context | Test / draft conversation | Live / shared conversation |
| --- | --- | --- |
| Agent configuration | Current mutable draft | Latest published immutable agent version |
| Workflow tools | Current workflow development heads | Each workflow's latest live release; never pinned |
| Reference files | Current draft knowledge files | Knowledge frozen into the published agent version |
| Runtime files | Current live runtime files, overlaid by development/test runtime files at the same path | Current live runtime files only |
| Access | Owner only | Owner, plus organization members when shared; non-owners never see the draft |

Version pinning for workflow tools does not exist: live tools always track the workflow's latest release; publishing the workflow is what changes tool behavior. If the user asks to freeze a tool's version, say so plainly — even when framed as a condition — and still attach it (policy `latest`); never abandon the attachment over it.

Publishing the agent snapshots name, instructions, tool identities, descriptions, approval policies, and knowledge files; it moves the live pointer and restamps every existing live conversation to the new version. It does not snapshot the runtime model.

Publishing is a hard cutover: mid-turn live conversations are interrupted (the person's in-flight answer never arrives), unexecuted approval requests expire, and each conversation reopens on its next message running the new version. Time publishes accordingly and tell the user.

Runtime lifecycle changes do not require publishing:

```bash
cai agent update <agentId> --disabled --json
cai agent update <agentId> --enabled --json
cai agent delete <agentId> --confirm --json
```

Disable pauses runtime immediately; enable resumes it. Delete is soft: it revokes live access, requests best-effort termination of active sessions and pending approvals, keeps history but makes it inaccessible, has no CLI restore, and does not guard dependent workflows.

## Runtime usage boundary

The user's coding-agent subscription covers builder reasoning, but hosted agent tests and conversations consume Controller AI runtime usage from the organization's pool, and connected providers may add charges or side effects. Say so before starting a hosted test.

## 4. Verify the draft as an agent

Saving the draft updates the agent's Test environment — there is no separate deploy step. Test conversations run each workflow tool's DEVELOPMENT version.

```bash
cai agent test <agentId> --draft --message "<representative request>" --json
cai agent conversation history <conversationId> --json
```

Test requests that should and should not select each tool. Verify exact arguments, truthful use of results, errors, and knowledge behavior.

Also offer the agent's app page so the user can try the draft hands-on before publishing (outputs include `appUrl` when the app origin is known); their impression outweighs your transcript reading.

- For a workflow tool, correlate a side-effectful call with its terminal workflow execution ID.
- For a direct integration action, inspect persisted conversation action events/run result and the intended external effect. Direct actions do not create workflow execution IDs.

For a pending approval, inspect the exact request first:

```bash
cai agent conversation approve <conversationId> --request <requestId> --json
cai agent conversation deny <conversationId> --request <requestId> --message "<reason>" --json
```

The server resolves the tool from the permission request; do not pass a human-readable tool name, and do not approve merely to make a test continue.

After `cai agent remove-tool`, test one request that must no longer use the removed capability and one retained capability.

## 5. Publish in dependency order

For attached workflows only:

```bash
cai use <workflowId>
cai workflow status --json
cai workflow publish --name "<release>" --json
```

Publish a workflow only when its verified development state must go live and the user authorized it. A workflow publish already followed by a live agent changes that tool's live behavior immediately; do not republish the agent merely to pick it up.

Then publish the agent:

```bash
cai agent publish-preflight <agentId> --json
cai agent publish <agentId> --name "<release>" --json
cai agent versions <agentId> --json
cai agent test <agentId> --live --message "<representative request>" --json
```

`publish-preflight` is the arbiter of structural readiness, with blocker parity with the publish gate (missing/unpublished/unrunnable tool flows, broken connections). When it lists blockers, report them and stop. When `hasBlockers` is false and the user asked for a release, publish — except an error you actually watched happen (a failed Test turn, an errored tool call), which blocks even a green preflight; the shape or style of successful output never does.

Email verification is not a publishing prerequisite. An unclaimed agent-created account may publish; email claiming only raises its monthly usage cap and enables account recovery.

Versioned agent state follows one loop: edit the draft, read back, verify in Test, publish, verify Live. Name, instruction, tool identity/description, approval-policy, and knowledge-file changes reach the published version only on the next agent publish.

Published agent versions have no rollback — never promise to "revert" one. Recover forward: restore the wanted DRAFT and publish anew — for agents, reapply an earlier snapshot from `cai agent versions`; for workflow tools, `cai workflow history` + `cai workflow restore --state <stateId>` restores the dev DEFINITION (not record data), then republish.

The live-bound exceptions:

- Workflow tools resolve the workflow's latest live release per runtime session, so a later workflow publish changes live behavior without another agent publish. An in-flight turn or approval episode may finish on its cached prior release; verify with a fresh Live conversation after the session refresh.
- A direct action's connection and authorization grant are live security bindings, not frozen version data. Rebinding the grant can redirect Live immediately; failure, revocation, or detachment can block Live immediately. Detaching also edits the draft, so publish the agent afterward to remove the unavailable tool from the published tool list.
- Runtime files are not part of the agent snapshot. Live conversations read current live runtime files; Test additionally overlays development/test runtime files, with the development copy winning on a path collision.

Draft evidence does not prove the live agent. Test is owner-only; shared users always run the latest published agent version and never see draft instructions, tools, or knowledge.

## 6. Offer Slack and organization sharing

After a successful publish, offer the next steps unprompted:

- **Slack**: talking to the agent in Slack needs the Slack ↔ Controller AI connection plus per-agent Slack configuration (slug, channel), both set up in the product — point the user there and offer a walkthrough.
- **Sharing**: agents remain private unless the user explicitly asks for organization access:

```bash
cai agent visibility <agentId> --value organization --json
```

Organization members use the published version. Direct actions run through the owner's connected external account; usage comes from the organization's pool. Rebinding or revoking that live grant immediately affects published agents; return to private visibility to revoke access.

## Final report

State the agent ID/version, tools and confirmation policies, workflow execution IDs and direct-action conversation evidence, draft/live status, live connection bindings, organization access, and anything unverified.
