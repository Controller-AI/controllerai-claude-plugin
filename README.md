# Controller AI for Claude Code

Build, test, run, and publish Controller AI workflows and user-facing agents from Claude Code.

The plugin bundles eleven focused Controller AI skills and connects to the hosted Controller AI MCP endpoint over OAuth. Its `exec` tool accepts only exact `cai ...` commands; it is not a shell and cannot run arbitrary programs, pipes, or redirections.

## Test from a checkout

From the plugin repository root:

```bash
claude --plugin-dir .
```

Authenticate the `controllerai` MCP server when Claude Code prompts you, then ask Claude to inspect or build a Controller AI workflow.

To install the checkout through Claude Code's local marketplace flow:

```bash
claude plugin marketplace add . --scope local
claude plugin install controllerai@controllerai-plugin-marketplace --scope local
```

## Try it

- `List my Controller AI workflows and summarize which ones are published. Do not change anything.`
- `Create and test a draft workflow that accepts a name and returns a greeting. Do not publish it.`
- `Inspect my most recent failed Controller AI development run and explain the first actionable cause.`

If authentication needs to be renewed, open `/mcp` in Claude Code and authenticate `plugin:controllerai:controllerai` again.

## Requirements

- Claude Code
- A Controller AI account, or an agent account created during MCP authorization
- Network access to `https://mcp.getcontroller.ai/mcp`

Controller AI documentation: <https://docs.getcontroller.ai>

Source: <https://github.com/Controller-AI/controllerai-claude-plugin>

Privacy: <https://getcontroller.ai/privacy-policy>

Terms: <https://getcontroller.ai/terms-of-service>

Support: <hi@getcontroller.ai>

License: [Apache-2.0](./LICENSE)
