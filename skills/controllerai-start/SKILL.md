---
name: controllerai-start
description: Connect, orient, and route work across Controller AI agents and no-code automations. Use when starting a Controller AI coding-agent or MCP session, after `cai connect`, or when deciding whether an outcome should be a ready integration action, a workflow, an agent, or a combination.
---

# Start with Controller AI

## Transports

The `cai ...` examples are canonical grammar. Use either the CLI or the typed MCP tools for a task; never mix transports within one task.

MCP tool names are domain-first snake_case derived from CLI command paths (`cai flow input add` -> `flow_input_add`). Read [references/mcp-tool-map.md](references/mcp-tool-map.md) for every command and reviewed exception. MCP has no workflow pin: pass `workflowId` on every workflow-scoped tool. Long-running MCP tools take `waitSeconds` (default 60, maximum 120) instead of CLI wait or timeout flags.

Local CLI setup commands do not exist on MCP. Use `account_status` for `auth status` and `doctor`, `account_claim_start`/`account_claim_verify` for email claiming, and `guide_list`/`guide_get` for bundled skills. `cai connect`, root `cai signup`, `cai auth login` and other local auth mutations, `cai skills install`, `cai skills update`, `cai use`, `cai config reset`, and `cai connection request` remain local-only; `cai agent present` and `cai file share` stay available to the in-app builder while hosted MCP uses their equivalents below.

- **Connect:** Call `connection_connect_url`, show its `url` as a markdown link, ask the user to finish connecting, then confirm with `connection_list` using its `integration`.
- **Finished agent:** Call `agent_present` and relay the returned agent card, including its link.
- **File for the user:** Call `file_share` and relay its returned link; Claude Code users may also just read it from the terminal.

## Your account on MCP

Call `account_status` first. During OAuth, the user explicitly chose **My Controller account** or **Separate agent account**; do not infer that choice from browser state. In your first reply, say which account `account_status` reports they are using. If it is an unclaimed agent account, offer to claim it with their email at any time. Never ask for passwords—the claim uses the 6-digit code sent to their email. After a successful claim, tell them they can open the app URL from `account_status`, choose **Forgot password** with that email, set a password, and sign in to the same workspace.

## Confirm the setup

On the CLI, run:

```bash
cai doctor
```

For a new terminal assistant, `cai agent setup` detects Claude Code, Codex, and
Cursor, ensures login, and installs the best available plugin or MCP-and-skills
fallback; use `cai agent setup --status --json` for a read-only machine plan.
Use `cai doctor` to compare installed version stamps and see the exact update
command, and `cai docs [path]` (or `cai docs --list`) to read platform docs in
the terminal.

MCP: use `account_status`; local diagnosis, login, and `cai connect` do not exist.

On the CLI, if authentication is missing and the human already has an account, run `cai connect`. If no account exists, run `cai signup` to create a provisional workspace and start building immediately. An unclaimed signup has the same functional access as a free account, including workflow and agent publishing; its workspace is not automatically deleted, and it has a $1 monthly usage cap (`cai signup` prints the exact amount). Email claiming is optional: run `cai signup email <address>`, ask the human account owner for the 6-digit code delivered to their inbox (never guess it), then run `cai signup verify <code>` to raise the monthly cap to $5 and enable durable account recovery. If agent signup is disabled, ask the human to register at https://app.getcontroller.ai/register, then run `cai connect`.

MCP: the OAuth caller is already provisioned; use `account_claim_start` and `account_claim_verify` for optional email claiming.

On the CLI, if `cai doctor` reports missing, stale, or damaged managed skills, run `cai skills update`, then run `cai doctor` again. When the `CLI version` row says a newer cai is available, run `cai upgrade` before continuing — it prints the explicit CLI install command and updates detected skills, and new versions carry fixes the rest of these skills assume. Skill updates are setup maintenance, not a step to repeat during every build.

MCP: use `guide_list` and `guide_get`; skill installation, updates, and CLI upgrades are not hosted-session steps. Never ask the user to paste a password, provider token, or Controller AI credential into chat.

Rename or change the default with `cai connection update <id> [--name "<displayName>"] [--default|--no-default]` (MCP: `connection_update`); delete only intentionally with `cai connection delete <id> --yes` (MCP: `connection_delete`).
Deleting a connection immediately removes authorization from bound nodes and agent tools.

## Route the outcome

Identify the deliverable before mutating anything:

| Outcome | Load next |
| --- | --- |
| A user-facing agent that can converse and act | `controllerai-agents` |
| One ready app operation attached to an agent | `controllerai-integration-actions` and `controllerai-agents` |
| A multi-step, triggered, stateful, or reusable automation | `controllerai-workflows` plus the relevant workflow skills |
| An agent with orchestrated capabilities | `controllerai-agents`, then both integration-action and workflow skills as needed |

Prefer a ready integration action when one existing operation completes the capability. Use a workflow when the behavior needs multiple steps, transformations, branches, loops, durable data, triggers, custom typed input/output, or reuse outside the agent.

If the user's outcome does not reveal whether they want an agent or an automation that runs on an event, ask one short clarifying question. Otherwise proceed with the smallest suitable surface.

## Lock targets before mutation

Before the first write, resolve every existing target against the effective workflow. Prefer an explicit ID or an object already connected to the pinned workflow. When a list is account-wide, filter it by exact name or ID; never take the first, latest, or similarly named result. Read back the chosen object's ID, name, and owning workflow or flow. Keep those IDs for the task and never rediscover the target by name.

Check `ok` and the complete `data` object from every mutation before projecting fields. A missing projected field is not proof that the mutation failed. Read back the returned ID or list the exact target before retrying. Never repeat a create until readback proves the first call created nothing.

## Platform docs

The platform reference lives at https://docs.getcontroller.ai. Fetch https://docs.getcontroller.ai/llms.txt for the machine-readable index of every page; append `.md` to any page path for raw markdown (for example https://docs.getcontroller.ai/start/safety.md), or fetch /llms-full.txt for the whole corpus at once. Skills remain the procedural authority — consult the docs for what they do not carry: the execution and cost models, type-system and version-rail contracts, the full CLI and expression-operation references, and the workflow API. When a docs page and a skill disagree, prefer the skill and its live platform queries (`cai expr ops`, `cai integration actions`).

## Keep rails explicit

- Agent Test mode uses the owner's mutable draft, current workflow development heads, current draft knowledge, and live runtime files overlaid by development/test runtime files. Live mode uses the latest published agent version, each workflow's latest live release, frozen published knowledge, and live runtime files only.
- Agent draft edits require another agent publish to reach Live. Publishing restamps existing live conversations; shared users always receive the latest published version and never the draft.
- Workflow publishes and direct-action connection/authorization grants are live bindings: a rebind can redirect Live, while revocation or detachment can block Live without another agent publish.
- Coding-agent subscriptions cover builder reasoning, not Controller AI hosted-agent runtime usage.
- Provider actions can send messages, mutate or delete records, and incur provider charges. Decide the per-action confirmation policy before attaching one.
- Never publish, share with the organization, replay a live trigger, or perform a consequential external test without clear user intent.
- A missing business rule is a blocker, not permission to infer one from names or sample data. Ask the user and leave the affected state unchanged.
- Stop when the requested state and its required evidence are reached. Do not add objects, publish, or keep probing beyond that boundary.
- When a platform limitation blocks the outcome, STOP and report the partial result with the blocking error; never keep probing until the clock expires.

At completion, report what was created, what is still draft or live, the evidence actually observed, and any connection or user decision still required.
