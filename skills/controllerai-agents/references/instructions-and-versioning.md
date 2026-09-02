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
- Do not retry more than once.
- If a connection is broken, name the integration that needs attention.
```

Delete any sentence that does not change behavior. Do not expose workflow, flow, node, rail, prop, schema tier, or version internals to the runtime user.

## Draft and live model

- Agent edits change only the mutable draft.
- Publishing snapshots display name, instructions, tool identities, descriptions and approval policies, and knowledge files into a new immutable version. The runtime model is not part of that snapshot.
- Publishing moves the agent's live pointer and restamps every existing live conversation to the new version. New and existing shared conversations therefore use the latest published agent version.
- Non-owners never receive draft instructions, tools, or knowledge. Test/draft conversations are owner-only.
- There is no rollback command; correct a bad version by editing the draft and publishing forward.
- Test/draft conversations use the current agent draft. Workflow tools run against each workflow's current development head.
- Live/shared conversations use the latest published agent version. Workflow tools follow each workflow's latest live release and cannot be pinned to an agent version.
- Direct action identity, description, and confirmation policy are captured in the agent version. Its connection and authorization grant remain live-bound: rebinding can redirect an invocation, while failure, revocation, or detachment can block it immediately without publishing the agent. Detachment still changes the draft tool list, so publish afterward to remove that tool from the published list.
- Organization members use the agent owner's connected external accounts for direct actions. Controller AI usage comes from the organization's shared pool.
- Workflow and agent publishing are independent. Publish only attached workflows whose verified development changes must become live. A workflow publish changes every live agent that follows it without another agent publish. Live workflow resolution is cached per runtime session: an in-flight turn or approval episode may finish on the prior release, then the session refreshes. Use a fresh Live conversation for post-publish proof and disclose any refresh warning.
- Development and live workflow records are separate. Stateful behavior needs live-data verification.

## File context by mode

- **Live/shared:** frozen knowledge from the latest published agent version plus current live shared/runtime files.
- **Test/draft:** current draft knowledge plus current live shared/runtime files overlaid by development/test shared/runtime files. On the same sandbox path, the development/test file wins.
- Runtime files are maintained outside the immutable agent snapshot. A live runtime-file change can therefore appear in live conversations without publishing the agent.

Do not describe Test and Live as sharing one mutable file context. Knowledge is versioned with the agent; runtime files are environment-aware live state.

## Updating an agent

For name, instructions, tool identity/description, tool approval policy, or knowledge-file changes:

1. Edit the mutable draft.
2. Read it back and verify it in a Test/draft conversation.
3. Publish any changed workflow dependency that should become live.
4. Run agent preflight and publish the agent.
5. Verify with a new Live conversation.

Do not republish the agent merely to make a followed workflow's new live release or a direct-action grant/connection rebind, failure, revocation, or detachment take effect; those effects are already live. After detaching an action, still publish the agent to remove the now-unavailable tool from its published tool list. Verify and disclose every live-bound change.

## Verification matrix

| Layer | Evidence |
| --- | --- |
| Workflow tool | Completed development `cai run flow` execution; an intentional `cai run flow <flowId> --target live --confirm-live` execution is stronger evidence of published behavior than a draft-agent proxy |
| Direct action | Persisted conversation action/run events, returned result, and intended external effect |
| Draft routing | Draft conversation chooses the correct tool and arguments |
| Published workflow | `workflow status` reports the intended live release and active state |
| Published agent | `agent publish` version plus a new live conversation |
| Stateful live behavior | Live conversation or trigger run reads/writes expected live records |

MCP: `run_flow` takes `waitSeconds`; pass any input object inline in `inputs`.

Direct actions do not create workflow execution IDs. Never claim a higher layer from only lower-layer evidence.
