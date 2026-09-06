---
name: controllerai-workflows
description: Build, inspect, repair, test and publish Controller AI workflows: typed callable flows, deterministic actions, branching, durable workflow data and triggers. Use for automations or an agent-callable application backend.
---

# Controller AI workflows

A workflow groups **flows**, **custom schemas**, **data collections** and **triggers**. A flow is one callable function with typed inputs and outputs; its connected nodes implement the function. Several flows can share the workflow's data store and call one another.

Flows are useful as API-like tools for agents. For example one “Investor CRM” workflow can expose create/get/update/delete/search flows for contacts, companies, deals and activities, plus a contact history view. An agent chooses a tool; the flow enforces its contract and performs the work.

## Start with the contract

Define each flow's inputs, outputs and failure behavior. Keep one operation per agent tool when that makes selection clearer than a giant operation dispatcher. Choose useful names and descriptions: “Search contacts by name, owner, tags and status” tells an agent when to call it.

Create the workflow, then explicitly create the first flow. `workflow_outline` provides a compact inventory; `flow_get` provides a flow's contracts, graph and configured values. Workflow IDs, flow IDs and node IDs are different identities.

Before changing an existing workflow, read its outline and the affected flow. Repair the existing IDs instead of creating replacements. For a substantial rewrite, save a named checkpoint. Restore rolls back the definition, not record changes or external effects.

## Assemble and test

1. Create any custom schemas and flow inputs/outputs first.
2. Discover actions and add nodes in execution order. Configure the type or action selector before dependent props; reload props if the selector changes.
3. Bind expressions using current `expr_context` names. Wire the source output to the destination so it appears in `inputs`.
4. Use a Return node to map promised flow outputs. Merely declaring an output does not populate it.
5. Test a complete development path with explicit representative input. Check the returned contract and same-environment data changes.
6. Apply the proven pattern to additional paths. Batch independent configuration when the available tool supports it; do not combine operations whose IDs depend on earlier responses.
7. Layout each changed flow, validate the saved workflow, and re-read its outline.

Use `controllerai-node-config` for action configuration, `controllerai-expressions` for computed values, `controllerai-flow-control` for routing/loops, `controllerai-data` for storage and `controllerai-runs` for execution evidence.

## Files and agents

A chat attachment is handled by a user-facing agent. For CSV enrichment, make a per-row flow accepting `investorName` and optional `company`, returning normalized contact information and evidence. The agent reads and writes the CSV. The native Create File action can generate files; it does not imply a native CSV reader/upload trigger.

For an agent tool, attach the tested flow with `agent_tool_workflow_add`. Its flow inputs and outputs are the tool contract. For unattended operation, add a trigger and map its event into these same inputs.

## Release and recovery

Graph edits affect development. Publishing creates the workflow's live release; attached live agents follow its latest release without another agent publish. The development/live record stores remain separate.

Use the user's existing authorization when deciding whether to publish. If publication is authorized, fix validation errors, publish, then verify Live. For a draft-only request, deliver the tested draft and its link.

On a state conflict, re-read the current saved version and reapply the intended change. Do not overwrite a newer graph with an old document. Report validation issues by their practical effect and inspect empty results where data was expected.

Detailed CLI/MCP sequences and lifecycle operations: [command reference](references/command-reference.md). Multi-domain command lookup: [command map](references/command-map.md).
