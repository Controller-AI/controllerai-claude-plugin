---
name: controllerai-start
description: Understand Controller AI and choose, build, test, or operate its agents, workflows, integrations, triggers and workflow data stores through MCP or the cai CLI. Use at the start of a Controller AI task, especially when a user describes an outcome without naming the right platform surface.
---

# Build with Controller AI

Controller AI is a platform for agents that do work and reusable automations that those agents can call. You are the builder: create and operate the user's artifacts through the connected tools. A **Controller AI agent** is the user-facing assistant you build, not you.

## What the platform can do

| Building block | What it provides | Choose it for |
| --- | --- | --- |
| Agent | Chat, reasoning, uploaded files, knowledge, integration actions and workflow tools | An assistant a person talks to, including “drop this CSV here and enrich it” |
| Ready integration action | One existing operation in an external app or native service | Search the web, read a sheet, create a CRM contact; attach directly to an agent when sufficient |
| Workflow | A container of reusable typed flows, shared schemas and durable data | Deterministic processing, business rules, reusable APIs, scheduled or event-driven work |
| Flow | One callable contract: typed inputs, connected nodes, typed return values | `researchInvestor(name, company)`, `createContact(...)`, `searchContacts(...)` |
| Nodes and expressions | Actions, calculations, branches, loops and subflow calls | Transform and route values without writing an application server |
| Workflow data store | Custom schemas and persisted records, with separate development and live stores | A CRM, tracker, knowledge index, audit log or application state |
| Trigger | Schedule, webhook, email or integration event mapped into a flow | Run an automation without a chat message |

An agent can call several flows in one workflow. Their inputs become tool arguments and their flow outputs become tool results. The data store is shared by that workflow's flows on the selected environment. Reusable flow contracts make a workflow an agent's application backend.

## Choose the deliverable from the experience

Use the user's intended interaction, even when they casually call everything a “workflow.”

- **File dropped into chat → enriched file back:** build an agent plus a per-record enrichment flow. The agent reads the attachment, handles rows and CSV formatting, calls the flow, and shares the resulting file. A workflow's `file` value is hosted-file metadata; it is not a native chat attachment upload or CSV parser.
- **CRM for an agent:** build a workflow containing typed CRUD/search flows over custom data schemas. Add a conversational agent only if requested or needed for the interaction.
- **One app operation:** attach the discovered action directly to the agent. A workflow wrapper is useful when it adds validation, transformation, persistence or a stable contract.
- **Several steps on an event:** build a workflow and trigger. Add an agent when a human also needs to ask questions or direct the process.

Make reasonable routine choices from context. Ask for missing information only when it changes the intended result; meanwhile finish independent work. A limitation in one surface can often be solved by combining the platform's surfaces. Explain a necessary change in architecture and keep pursuing the original outcome.

## Orient once, then build

Use `account_status` to identify the connected workspace. MCP is already authenticated and has no local workflow pin: pass the returned `workflowId` on scoped calls. Do not ask an authenticated user to sign up again. Account claiming is optional and relevant to recovery or usage limits, not a build or publishing prerequisite.

For an existing artifact, resolve its exact ID and inspect its outline or tool list. Keep returned IDs during the task. For a new artifact, create it and use its returned ID; inspect `flowCount` and `nextStep`: reuse a returned default flow, or create the first flow when the workflow is empty.

Use the tools actually available in the session. MCP takes inline JSON/text; CLI examples take files where shown and accept `--json` for machine-readable output. The [tool map](references/mcp-tool-map.md) maps the CLI grammar to MCP names and exceptions. There is no need to perform local CLI installation to use connected MCP.

## Build, verify and deliver

1. Define the experience and a small representative test. For a complex API, decide schemas, inputs, outputs and error behavior before assembling nodes.
2. Discover capabilities and current schemas. Use exact action keys and the current expression bindings rather than guessing names.
3. Build a complete path, then test its actual outputs and stored or external effects. Expand the proven pattern to the remaining paths.
4. Validate the graph, inspect suspicious runs, and re-read saved state. A green run alone does not prove returned data or persistence is correct.
5. Publish when the user's request authorizes a live result. Explicit authorization persists; do not ask again just because publishing is the next step. Verify the live result after publication.
6. Return product links, the observed result, environment and any concrete limitation. Remove only scratch artifacts you created for abandoned attempts.

Development graph edits are separate from the published live graph. Publishing does **not** copy development records into the live data store. Agents have their own draft/published versions; agent Test calls development workflows, while Live calls each workflow's latest live release.

For mutation errors or large-result downloads, inspect the complete result and the target state before retrying a create. Missing projected fields do not prove failure. Never treat an unknown outcome as permission to repeat a write.

## Go deeper when needed

Load the relevant skill at the point of use: `controllerai-agents`, `controllerai-workflows`, `controllerai-data`, `controllerai-expressions`, `controllerai-node-config`, `controllerai-flow-control`, `controllerai-triggers`, `controllerai-runs`, `controllerai-ai-nodes`, or `controllerai-integration-actions`. Each explains its capability and links detailed command examples. Platform reference: https://docs.getcontroller.ai/llms.txt; hosted current guides: `guide_list` and `guide_get`.
