<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Workflow command families

Read the family you need; `cai <domain> --help` lists flags, and the [generated tool map](../../controllerai-start/references/mcp-tool-map.md) gives the MCP tool and parameter names.

## Lifecycle

```text
cai workflow list|create|get|rename|set-description|duplicate|delete
cai workflow outline|document|state|status|versions|releases|history|save|restore|publish
```

`outline` is the compact inventory; `state --flow <flowId>` adds node and edge identities; `document` is the complete JSON. `save` creates a named checkpoint, `restore --state <stateId>` rolls the definition back, `publish` creates the live release and refuses validation errors unless the user has accepted them explicitly. `delete --yes` undeploys triggers and soft-deletes; it is refused while a draft or live agent depends on the workflow.

## Canvas

```text
cai flow list|get|create|rename|duplicate|delete|reorder|layout
cai flow input add|update|delete
cai flow output add|update|delete
cai integration search|actions|action
cai node add|status|get|props|set|connect|disconnect|delete
```

## Composition

```text
cai node only-when
cai branch list|add|update|delete|connect|reorder
cai flow for-each
cai node set-flow
```

Router and If/Else outputs connect through the branch commands; generic `node connect` handles only `main` and `error` outputs.

## Verification

```text
cai run inputs|node|flow|discover
cai exec list|get|flow|node|tree
```

`run discover` executes the real action in a scratch flow and can cause third-party effects or charges; do not use it casually on mutating actions.

## Other domains

```text
cai connection list|update|delete
cai trigger list|get|create|search|replay|test-events
cai schema list|create|update|delete
cai data collections|query|create|update|delete|import
cai agent create|get|list|update|add-workflow|add-action|add-file|remove-tool|publish|conversation
```

Each family has its own skill; the map above covers CLI-to-MCP names for all of them.
