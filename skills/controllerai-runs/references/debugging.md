<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Execution debugging

Use this order; it avoids changing a downstream node when the defect is upstream.

## 1. Confirm the version and the graph

- `workflow_status` — `workflowId`
- `flow_get` — `flowId`, `workflowId`

Node runs use development. Flow runs use development unless the published version was selected explicitly. Every run can call real providers.

## 2. Find the first divergence

- `execution_tree` — `workflowExecutionId`, `cursor`, `flowId`, `limit`, `status`, `workflowId`

Follow `nextCursor` as `cursor`; use `status` (an array) or an exact `flowId` on a large run. Classify each node as completed, completed with a handled error, failed, cancelled, paused, skipped on purpose (`only_when_false`), skipped unexpectedly (`upstream_skipped` behind a failure or a wrong branch), or absent. The first divergence is usually the cause; later failures are consequences.

## 3. Inspect the node boundary

- `execution_node_get` — `nodeExecutionId`, `full`, `workflowId`

In order:

1. Resolved configs: did each expression resolve, and to the right type and value? A null entry on a dynamic prop that the contract did not intend is the failure even when the node is green; read each entry's `resolutionWarnings`.
2. Port: did the node select the branch or output the test expected?
3. Output: did the component return the promised shape and values? Compare a flow's return with its declared outputs.
4. Status and error: did the provider or the engine reject the call?
5. Upstream rows and wiring: did every expected producer run and connect to this node?

## 4. Fix by symptom

| Symptom | Inspect next |
| --- | --- |
| Missing or wrong input | Wiring, branch connection, or the test input ID |
| Null dynamic config | Expression binding, field path, or a missing source value |
| Invalid option or account | Node connection, dependency-ordered props, or a stale option |
| Provider error | Integration configuration, credentials, quota, or payload |
| Green node, wrong output | Component choice, output contract, or prompt |
| Unexpected skip | Guard, branch condition or order, or an upstream failure |
| Empty record query | Store (development versus live), field key, optional-filter semantics |
| Provider authentication failure | Connection identity and the provider's error; an active status is not a fresh credential test |
| Return value empty | Return node bindings, skipped branches, or resolved empty values |

Re-read metadata and contexts before editing. Rerun the smallest failed scope, then the full flow.

## 5. Avoid false confidence

- A compile pass does not prove runtime data exists.
- A node pass does not prove downstream mapping: a node run does not execute wired upstream nodes.
- A development pass does not prove live data or the live release state.
- A top-level completed status does not prove every intended branch ran, and a loop's success does not prove every iteration.
- Keep the evidence of failed runs and separate it clearly from the final verified result; broaden the test when a fix changes shared behavior.
