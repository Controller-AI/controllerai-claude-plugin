# Controller AI for Claude

Controller AI is an automation platform built for AI agents to operate and for people to understand. Ask Claude to build, test, publish, diagnose, and maintain durable automations using 1,000+ integrations. Each automation remains visible as deterministic steps on a visual canvas, with inspectable execution and version history and connections kept in one place. Building user-facing agents is also supported, but automation is the primary focus.

The plugin works in Claude Code, Claude Cowork and Claude chat. It bundles twelve focused skills, including one for answering company questions from the company brain, and connects to the hosted Controller AI MCP server, which authenticates through Controller AI OAuth and exposes only typed tools: one per supported action, with input and output schemas plus read/write annotations. Nothing is installed locally.

## Claude Code

From the plugin repository root, test the checkout with:

```bash
claude --plugin-dir .
```

To install the checkout through Claude Code's local marketplace flow:

```bash
claude plugin marketplace add . --scope local
claude plugin install controllerai@controllerai-plugin-marketplace --scope local
```

Authenticate the `controllerai` MCP server when Claude Code prompts you. If authentication needs to be renewed, open `/mcp` and authenticate `plugin:controllerai:controllerai` again.

## Claude Cowork

Install the plugin from its GitHub repository. Installing the plugin loads its skills but does not grant access to your Controller AI account by itself. To connect:

1. Open the Controller AI plugin details.
2. Open **Connectors** and select **Connect** for `controllerai`.
3. Complete Controller AI OAuth: sign in to your Controller AI account, or sign up for one, and approve the connection.
4. Start a fresh task and ask Claude to use the Controller AI connector.

## Try it

- `Review the recurring tasks, handoffs, and tools I describe. Identify the best automation opportunity, then build and test it in Controller AI.`
- `Create and test a draft automation that accepts a name and returns a greeting. Do not publish it.`
- `Inspect my most recent failed Controller AI automation run, explain the first actionable cause, and safely fix the draft.`

## Requirements

- Claude Code, Claude Cowork, or Claude chat on a paid plan
- A Controller AI account: sign in, or sign up for one, during MCP authorization
- Network access to `https://mcp.getcontroller.ai/mcp`

Controller AI documentation: <https://docs.getcontroller.ai>

Source: <https://github.com/Controller-AI/controllerai-claude-plugin>

Privacy: <https://getcontroller.ai/privacy-policy>

Terms: <https://getcontroller.ai/terms-of-service>

Support: <https://getcontroller.ai/support> or <hi@getcontroller.ai>

License: [Apache-2.0](./LICENSE)
