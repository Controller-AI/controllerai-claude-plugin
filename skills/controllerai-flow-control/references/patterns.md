# Flow-control patterns

## Guard a side effect

Use `only-when` when false should mean "skip this action" and nothing else:

```text
Read record -> Send notification [only when status changed]
```

Test both true and false. A false run should show an intentional skip reason.

## Exclusive decision

Use If/Else when only one outcome must occur:

```text
Score -> If/Else
           high priority -> Escalate
           normal        -> Queue
           Else          -> Record unknown
```

Put the most specific/highest-priority conditions first. Never write a complement condition for Else.

## Independent fan-out

Use Router when every applicable action should run:

```text
New order -> Router
              international -> Customs review
              high value    -> Fraud review
              gift          -> Gift note
```

One order may take all three routes. If that is not acceptable, use If/Else.

## Per-item subflow

Use `cai flow for-each` to create the item flow, typed item input, loop node, and list wiring atomically. First run `cai node get <sourceNodeId> --json`, choose the intended `outputs[]` entry, and pass its exact opaque `id` as `--source-output`; never pass its label or assume the first output. Omit list-selection flags when the whole selected output payload is a list; use `--list-field <nested.path>` for a list below `.data`; reserve `--list-js` for an exact advanced expression or `--js-file <path|->` for that expression in a UTF-8 file/stdin. The three selection forms are mutually exclusive.

The runtime limit defaults to 500 items and supports 1 through 10000. Use `--max-items` when the intended bound differs, then verify the returned read-back. Put meaningful work inside the generated body flow and declare body outputs before consuming aggregate results.

Decide explicitly whether one child-flow failure should stop the sequential loop. Use `--continue-on-error` only when partial completion is acceptable and downstream logic handles failed iteration records. It does not recover a list or mapping expression that fails before an iteration starts.

## Error path

Continue-on-error is for steps whose failure must not block the main path — notifications, logging, optional enrichment. Enabling it captures the node's error, downstream nodes still execute (receiving null for the failed node's output), and the run ends `completed_with_error`:

```bash
cai node continue-on-error <nodeId> --enable --json
```

Wiring the Error output is OPTIONAL — add it when a failure needs explicit handling, not as a precondition:

```bash
cai node connect --flow <flowId> --from <nodeId> --to <handlerNodeId> --output error --json
```

Never enable continue-on-error on critical-path steps (validate → charge → confirm must fail fast), and never merely to make the parent execution green. Read back the setting after changing it.
