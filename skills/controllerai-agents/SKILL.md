---
name: controllerai-agents
description: Build Controller AI user-facing assistants with chat and uploaded files, instructions, knowledge, direct integration actions and workflow tools. Use for conversational experiences, file enrichment, tool selection, testing and publishing agents.
---

# Controller AI agents

A Controller AI agent is an assistant a person can chat with. It reasons, reads conversation attachments, uses reference knowledge, calls tools, and creates files for the user. Give it direct integration actions for individual app operations and workflow tools for reusable processing or data access.

The agent has a mutable draft and published versions. Test uses the draft; shared users use the latest published version. Disabling an agent pauses runtime independently of publication.

## Design the experience

Write instructions that state the agent's role, required information, how to choose tools, how to handle missing data, and how to present results. Put deterministic validation and database changes in workflow tools. Put stable reference material in knowledge files. Avoid telling an agent to retry all failures indiscriminately: reads, rejected validation and uncertain mutations require different responses.

For a workflow tool, describe when to use it, its inputs and its result. For a ready action, discover the exact integration/action and use a valid connection. Attach with `agent_tool_workflow_add` or `agent_tool_action_add`; inspect `agent_tool_list` afterward.

Confirmation policy is a tool setting, not the builder's permission to finish authorized work. Choose it for the intended user experience. A read/search tool normally runs directly; consequential unattended actions should match the user's stated scope.

## Recipe: upload a CSV, enrich investors, download the result

1. Build a flow `Research one investor` accepting a person's name and optional company. Return structured fields such as matched name/company, work email, public contact links, source URLs, match confidence and a per-row error/status.
2. Test one clear match and one ambiguous/missing match. Do not invent an email or claim none exists merely because search did not find one.
3. Create the agent and attach that flow. Instruct it to inspect the CSV columns, preserve original columns and row order, process every row, keep failures on their original rows and append the enrichment columns.
4. The **agent** reads the uploaded file and produces the CSV. Workflows receive typed values extracted from rows. A `file` workflow value is hosted-file metadata, not a chat upload or parser.
5. Test with a small CSV including quoted commas, a missing company, an ambiguous name and a failed lookup. Verify the exported file opens and retains every input row.
6. Share the generated attachment through the supported file-sharing surface. A message saying “saved the file” is not proof of an accessible download.

## Files have distinct roles

- **Conversation attachments:** files supplied or generated during a chat, tied to that conversation.
- **Knowledge:** reference files in the agent draft; publishing freezes the live knowledge snapshot.
- **Shared runtime workspace:** files the agent can work with across its runtime. Shared-file writes affect the live workspace; Test sees live files overlaid by development files. They are not a substitute for uploading a file to a conversation.

Use `agent_conversation_file_add` with inline UTF-8 content to attach CSV/text, `agent_conversation_file_list` to discover attached and generated files, `agent_conversation_file_read` for text, and `agent_conversation_file_download` to return the original file through a private download link. CLI equivalents are `cai agent conversation add-file|files|read-file|download-file`.

Use returned file metadata and paths. Runtime mounts can prefix workspace paths; rediscover the actual attachment or workspace file instead of inventing a new path.

## Verify conversations and release

Create a draft conversation and keep its ID for follow-up. `agent_test` creates a new conversation; use conversation send/status/history for subsequent turns. History is a compact transcript by default: use `limit` and `before: nextBeforeId` for older messages, or `last: true` for the latest reply. Request `raw: true` only for SDK diagnostics. File results are discoverable by conversation ID without reading raw events.

Verify tool selection, arguments, terminal results, persisted data and downloadable outputs. Resolve a real approval request only within the user's authorized scope and in the same conversation; sending the original prompt again creates duplicate work.

When release is authorized: publish changed workflow dependencies, run agent preflight, publish the agent, and test a fresh Live conversation. Live workflow tools follow the latest published workflow, while Test uses its current development head. Present the returned agent link or `agent_present` result. Organization sharing is separate from publishing and should match the requested audience.

Command sequences: [reference](references/command-reference.md). Exact version/file semantics: [instructions and versioning](references/instructions-and-versioning.md).
