# Workflow command map

Read only the sections relevant to the requested work. Use `cai <domain> --help` as the authoritative flag reference.

## Setup and workflow lifecycle

```text
cai connect
cai doctor
cai config reset [--local|--global]
cai auth status
cai workflow list|create|get|rename|set-description|duplicate|delete
cai workflow state|status|versions|releases|history|save|restore|publish
cai use <workflowId>
```

`workflow save` creates a named development checkpoint. `workflow publish` promotes development to live. Version history may be plan-gated. `workflow delete --yes` undeploys triggers and soft-deletes the workflow row; it refuses while any draft or LIVE agent depends on the workflow. Detach draft tools with `agent remove-tool`, publish detached drafts that are still LIVE, then retry. `--force` is deprecated and inert. Deletion is not a permanent history purge, and deleted rows currently have no CLI restore command.

## Canvas and graph

```text
cai flow list|get|create|rename|duplicate|delete|reorder|layout
cai flow input add|update|delete
cai flow output add|update|delete
cai node find|search|components|component
cai node init|status|get|props|set|connect|disconnect|delete
```

Declare flow outputs before adding a Return response node. Its props are derived from those outputs.

`cai flow input add <flowId> --display "<human label>" --type <type>` returns the compiler-current `jsBinding`. Use the latest value from `flow get` (`jsBinding`) or `workflow outline` (`binding`), re-read it after every input rename, and never cache its spelling across renames. `legacyBinding` is diagnostic-only.

## Conditions and composition

```text
cai node only-when
cai branch list|add|update|delete|connect|reorder
cai flow for-each
cai node set-flow
```

Use branch-specific connection commands for Router and If/Else outputs. Generic `node connect` supports only `main` and `error` outputs.

## Verification

```text
cai run inputs|node|flow|discover
cai exec list|get|flow|node|ports|tree
```

`run discover` executes the real integration action in a scratch flow and can incur third-party effects or usage. Do not use it casually for mutating actions.

## Product surfaces

```text
cai connection list|connect
cai trigger list|get|search|props|options|create|events|event|select-event|replay|toggle|rename|delete
cai schema list|get|resolve|create|update|delete
cai data collections|fields|query|get|create|update|delete|delete-many|import|copy-to-live
cai agent list|get|create|update|add-workflow|remove-tool|add-file|tools|publish-preflight|publish|versions|visibility
cai agent conversation list|create|status|history|send|approve|deny
```

Connections are authorized in a browser. The CLI returns and opens the Controller AI connection URL.
Agent visibility supports only `private` and `organization`; organization is an explicit live access change after agent publish.
