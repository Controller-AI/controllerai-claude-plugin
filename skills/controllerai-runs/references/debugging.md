# Execution debugging

Use this order; it avoids changing a downstream node when the defect is upstream.

## 1. Confirm rail and graph

```bash
cai workflow status --json
cai flow get <flowId> --json
```

Node runs use development. Flow runs default to development; `cai run flow <flowId> --target live --confirm-live` runs the published version. Trigger replay also defaults to development and uses the same flags for live. Every rail can call real providers and create external effects.

MCP: `run_flow` takes `waitSeconds` and may return before the execution is terminal.

## 2. Find the first divergence

```bash
cai exec tree <workflowExecutionId> --limit 50 --json
```

Follow `nextCursor` with `--cursor`, and use repeatable `--status` or `--flow <flowId>` when the
run is large. Classify each node as completed, failed, skipped intentionally, skipped unexpectedly,
or absent. The first divergence is usually the cause; later failures are often consequences.

## 3. Inspect the node boundary

```bash
cai exec node <nodeExecutionId> --full --json
```

Check:

1. Resolved configs: did each expression resolve, and to the correct type/value?
2. Output: did the component return the promised shape and value?
3. Status/error: did the provider or engine reject the call?
4. Upstream execution rows and wiring: did every expected producer run and connect to this node?

## 4. Fix by category

| Evidence | Likely surface |
| --- | --- |
| Missing/wrong input | Wiring, branch connection, or test input ID |
| Null dynamic config | Expression binding, field path, or missing source value |
| Invalid option/account | Node connection or dependency-ordered props |
| Provider error | Integration configuration, credentials, quota, or payload |
| Green node, wrong output | Component choice, output contract, or prompt |
| Unexpected skip | only-when, branch condition/order, or upstream failure |

Re-read metadata/context before editing. Then rerun the smallest failed scope and finally the full flow.

## 5. Avoid false confidence

- A compile pass does not prove runtime data exists.
- A node pass does not prove downstream mapping: `run node` does not execute wired upstream nodes. Require `run flow` for any `inputs.<name>` expression.
- A dev pass does not prove live data or live release state.
- A top-level completed status does not prove every intended branch ran.
