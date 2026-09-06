---
name: controllerai-runs
description: Test and debug Controller AI workflows, flows and nodes using execution history, inputs, resolved configuration, ports, output values and record effects. Use to prove a build works or investigate a failed or suspicious result.
---

# Prove workflow behavior

Execution tools show what the platform actually ran: inputs, node configuration, output values, selected ports, skips and errors. Use them to verify contracts and effects, not just a green status.

## Select the right test

- Test a **node** to establish its configuration, provider connection or output shape. Node tests use development state.
- Run a **flow** to verify connected steps and the return contract. Choose development or the published live version explicitly.
- Test an **agent conversation** to verify its tool choice and file handling, then correlate workflow calls with their executions.
- Replay a **trigger sample** into development to verify event-to-flow mapping before enabling live behavior.

Development and live use different data stores, but development provider actions can still perform real external operations. Test within the user's authorized scope and use synthetic data where appropriate.

## Run and inspect

Inspect required inputs, pass explicit representative values, and capture the returned execution ID. MCP runs support a bounded wait. A timeout or running status means inspect that execution later; it does not mean repeat the operation.

Start with the compact execution summary or tree. Drill into a failing, skipped or suspicious node to inspect its received inputs, resolved config and output. Check the expected port before moving downstream. Compare an actual returned value with the promised flow contract.

For data changes, query the affected records on the same environment. For files, verify the returned artifact and contents. For direct provider actions, verify the provider result or intended external state rather than expecting a workflow ID.

## Diagnose common failures

| Symptom | Inspect next |
| --- | --- |
| Missing required config | Selector choice, refreshed props, enabled fields and literal/expression mode |
| Unknown field/binding | Node expression context, exact connection output and current input binding |
| Empty query | Record environment, field key, optional-filter handling and matching semantics |
| Provider authentication failure | Connection identity and provider error code; saved active status is not a fresh credential test |
| Green run but no useful output | Return-node mappings, skipped branches and resolved empty values |
| Mutation result too large or uncertain | Download/reference receipt and exact current target state before retrying |
| Graph changed unexpectedly | Saved-state history and current node/edge identities; do not replace a newer graph with a cached copy |

Fix the cause and rerun the affected scenario. Broaden tests when the fix changes shared behavior. Keep failed-run evidence with a clear distinction between historical failures and the final verified result.

Commands: [reference](references/command-reference.md). Execution details: [debugging](references/debugging.md).
