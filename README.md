# Controller AI for Claude

Controller AI is an automation platform built for AI agents to operate and for people to understand. Ask Claude to build, test, publish, diagnose, and maintain durable automations using 1,000+ integrations. Each automation remains visible as deterministic steps on a visual canvas, with inspectable execution and version history and connections kept in one place. Building user-facing agents is also supported, but automation is the primary focus.

The plugin works in Claude Code and Claude Cowork. It bundles eleven focused skills and supports both the hosted Controller AI MCP connector and the local `cai` CLI. The hosted connector exposes only its 158 typed tools, one per supported action, with input and output schemas plus read/write annotations.

## Choose a connection path

Both paths are supported:

- **Hosted MCP connector:** works without a local CLI, authenticates through Controller AI OAuth, and exposes typed Controller AI actions directly. This is the normal path for hosted and no-install environments.
- **Local `cai` CLI:** often works best for local or file-heavy tasks and uses the Controller AI account already connected on the machine.

If both paths are available, Claude should respect your explicit choice. Otherwise it will identify the selected transport and account or context before the first Controller AI operation. It must not silently switch or mix transports or accounts during a task.

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

For the hosted path, authenticate the `controllerai` MCP server when Claude Code prompts you. If authentication needs to be renewed, open `/mcp` and authenticate `plugin:controllerai:controllerai` again.

For the local path, install the Controller AI CLI, run `cai connect`, and verify the selected account with `cai doctor`.

## Claude Cowork

Install the plugin from its GitHub repository. Installing the plugin loads its skills but does not grant access to the hosted Controller AI account by itself. To use hosted MCP:

1. Open the Controller AI plugin details.
2. Open **Connectors** and select **Connect** for `controllerai`.
3. Complete Controller AI OAuth, then choose **My Controller account** to sign in or sign up and connect that Controller AI account, or **Separate agent account** to create or continue without email or a password.
4. Start a fresh task and ask Claude to use the Controller AI connector.

Cowork may also find an already installed and authenticated local `cai` CLI. That is a valid path, but it exercises the local account rather than the hosted connector. Ask Claude to use a specific path when you want to test one of them explicitly.

## Try it

- `Review the recurring tasks, handoffs, and tools I describe. Identify the best automation opportunity, then build and test it in Controller AI.`
- `Create and test a draft automation that accepts a name and returns a greeting. Do not publish it.`
- `Inspect my most recent failed Controller AI automation run, explain the first actionable cause, and safely fix the draft.`

## Requirements

- Claude Code or Claude Cowork
- No Controller AI account is required in advance: during MCP authorization, choose **My Controller account** to sign in or sign up, or **Separate agent account** to create or continue without email or a password
- Network access to `https://mcp.getcontroller.ai/mcp` for the hosted path
- The Controller AI `cai` CLI for the optional local path

Controller AI documentation: <https://docs.getcontroller.ai>

Source: <https://github.com/Controller-AI/controllerai-claude-plugin>

Privacy: <https://getcontroller.ai/privacy-policy>

Terms: <https://getcontroller.ai/terms-of-service>

Support: <https://getcontroller.ai/support> or <hi@getcontroller.ai>

License: [Apache-2.0](./LICENSE)
