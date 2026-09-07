<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Agent instructions and versioning

## Instruction template

```markdown
You are <role> for <audience>. You own <outcome>.

## Actions

- `<tool_name>`: Use when <specific situation>. Pass <inputs in user terms>. It returns <result>; <how to use it>.

## Judgment

- <decision threshold or boundary>
- <what requires clarification or approval>
- <what must never happen>

## Responses

- <tone, length, format>

## When something fails

- Tell the user what failed and what you were trying to do.
- Retry a read once; do not retry a rejected validation; check the result before repeating an uncertain write.
- If a connection is broken, name the integration that needs attention.
```

Delete any sentence that does not change behavior. Do not expose workflow, flow, node, rail, prop, schema-tier or version internals to the runtime user.

## Draft and live model

- Agent edits change only the mutable draft; saving the draft updates Test immediately.
- Publishing snapshots display name, instructions, tool identities, descriptions and approval policies, and knowledge files into a new immutable version. The runtime model is not part of that snapshot.
- Publishing moves the live pointer and restamps every existing live conversation to the new version; new and existing shared conversations therefore use the latest published agent version. In-flight answers are interrupted and pending approvals expire.
- Non-owners never receive draft instructions, tools or knowledge. Test conversations are owner-only.
- There is no rollback; correct a bad version by editing the draft and publishing forward. `agent versions` is the release history.
- Test conversations run each workflow tool against its current development head; live conversations follow each workflow's latest live release and cannot pin a version.
- Workflow and agent publishing are independent. A workflow publish changes every live agent that follows it without another agent publish. Live workflow resolution is cached per runtime session: an in-flight turn or approval episode may finish on the prior release, then the session refreshes. Use a fresh live conversation for post-publish proof and disclose any refresh warning.
- A direct action's identity, description and confirmation policy are versioned; its connection and grant are live: rebinding can redirect an invocation, and failure, revocation or detachment can block it immediately. Confirmation is strictest-wins: turning it on for a direct action blocks the published action at once until the next publish carries it. Detachment also changes the draft tool list, so publish afterward to remove the tool from the published list.
- Organization members use the owner's connected external accounts for direct actions, and usage comes from the organization's shared pool.
- Development and live workflow records are separate; stateful live behavior needs live-store proof.

## File context by mode

- **Live:** knowledge frozen in the latest published agent version plus current live shared or runtime files.
- **Test:** current draft knowledge plus current live shared or runtime files, overlaid by development/test files at the same sandbox path, where the development/test file wins.
- Runtime files live outside the immutable snapshot, so a live runtime-file change appears in live conversations without a publish. Conversation attachments belong to one conversation only.

## Updating a published agent

1. Edit the draft.
2. Read it back and verify it in a Test conversation.
3. Publish any changed workflow dependency the user authorized.
4. Run preflight and publish the agent.
5. Verify with a new live conversation.

Do not republish the agent merely to pick up a followed workflow's new release or a direct-action rebind, failure, revocation or detachment; those effects are already live. Verify and disclose every live-bound change.

## Verification matrix

| Layer | Evidence |
| --- | --- |
| Workflow tool | A completed development flow run; an intentional, authorized live flow run is stronger evidence of published behavior than a draft-agent proxy |
| Direct action | Persisted conversation action events, the returned result, and the intended external effect |
| Draft routing | A Test conversation that chooses the correct tool with the correct arguments |
| Published workflow | `workflow status` reporting the intended live release and active state |
| Published agent | The publish version plus a new live conversation |
| Stateful live behavior | A live conversation or trigger run that reads and writes the expected live records |

Direct actions do not create workflow execution IDs. Never claim a higher layer from lower-layer evidence alone.

## File-handling agents

An agent, not a workflow, reads a chat-uploaded file and produces the output file; workflows receive typed values extracted from the rows, and a workflow `file` value is hosted-file metadata, not an upload or a parser. For a per-row enrichment: build and test a typed one-item flow (clear match, ambiguous match, missing data, a failed lookup), attach it, instruct the agent to inspect the columns, process every row, keep failures on their original rows, preserve column and row order and append the new columns, then have the exported file checked (quoted commas, a missing company, an ambiguous name) for opening and retaining every input row. The upload and the download happen on the product's file surface, which the builder cannot drive, so that check is the user's; "saved the file" is not proof of an accessible download.
