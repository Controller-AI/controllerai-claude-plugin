---
name: controllerai-start
description: Connect, orient, and route work across Controller AI agents and no-code automations. Use when starting a Controller AI coding-agent session, after `cai connect`, or when deciding whether an outcome should be a ready integration action, a workflow, an agent, or a combination.
---

# Start with Controller AI

Controller AI supports two command transports:

- In an MCP host, send each exact literal `cai ...` command through the single `exec` tool; never translate it into another grammar.
- In a local coding environment, use the installed `cai` CLI. It is often the better path for local or file-heavy work.

Respect an explicit user choice of transport or account. When only one transport is available, use it. When both are available and the user did not choose, select the one that best fits the task and briefly identify the selected transport and account or context before the first Controller AI operation. Do not mix transports or silently switch transports or accounts during a task. If the selected path becomes unavailable or loses authentication, stop and offer the other path instead of switching behind the user's back.

Do not call Controller AI REST or MCP routes directly.

## Confirm the setup

Run `cai doctor` through the selected transport.

For the hosted MCP path, authentication belongs to the bundled Controller AI connector. If it is unavailable or unauthenticated, stop and ask the user to connect or reauthenticate it in the host. Do not try to repair the hosted path by invoking a machine-local `cai`, `cai connect`, `cai signup`, `cai skills update`, or a package installer.

For the local CLI path, if authentication is missing and the human already has an account, offer `cai connect`. If no account exists, offer `cai signup` to create a provisional agent account and start building immediately. An unclaimed account has the same functional access as a free account, including workflow and agent publishing; it is not automatically deleted, and it has a $1 monthly usage cap (`cai signup` prints the exact amount). Email claiming is optional: run `cai signup email <address>`, ask the human account owner for the 6-digit code delivered to their inbox (never guess it), then run `cai signup verify <code>` to raise the monthly cap to $5 and enable durable account recovery. If agent signup is disabled, ask the human to register at https://app.getcontroller.ai/register, then run `cai connect`.

Only on the selected local CLI path, if `cai doctor` reports missing, stale, or damaged managed skills, offer `cai skills update`. If it reports a newer CLI, ask before installing or upgrading software; after approval, use the documented upgrade command and rerun `cai doctor`. Never install or upgrade the CLI automatically. Never ask the user to paste a password, provider token, or Controller AI credential into chat.

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

## Platform docs

The platform reference lives at https://docs.getcontroller.ai. Fetch https://docs.getcontroller.ai/llms.txt for the machine-readable index of every page; append `.md` to any page path for raw markdown (for example https://docs.getcontroller.ai/start/safety.md), or fetch /llms-full.txt for the whole corpus at once. Skills remain the procedural authority — consult the docs for what they do not carry: the execution and cost models, type-system and version-rail contracts, the full CLI and expression-operation references, and the workflow API. When a docs page and a skill disagree, prefer the skill and its live CLI queries (`cai expr ops`, `cai node find`).

## Keep rails explicit

- Agent Test mode uses the owner's mutable draft, current workflow development heads, current draft knowledge, and live runtime files overlaid by development/test runtime files. Live mode uses the latest published agent version, each workflow's latest live release, frozen published knowledge, and live runtime files only.
- Agent draft edits require another agent publish to reach Live. Publishing restamps existing live conversations; shared users always receive the latest published version and never the draft.
- Workflow publishes and direct-action connection/authorization grants are live bindings: a rebind can redirect Live, while revocation or detachment can block Live without another agent publish.
- Local coding-agent subscriptions cover builder reasoning, not Controller AI hosted-agent runtime usage.
- Provider actions can send messages, mutate or delete records, and incur provider charges. Decide the per-action confirmation policy before attaching one.
- Never publish, share with the organization, replay a live trigger, or perform a consequential external test without clear user intent.
- When a platform limitation blocks the outcome, STOP and report the partial result with the blocking error; never keep probing until the clock expires.

At completion, report what was created, what is still draft or live, the evidence actually observed, and any connection or user decision still required.
