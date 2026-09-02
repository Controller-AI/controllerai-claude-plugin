# Trigger lifecycle

## State sequence

```text
source decision
  -> exact node/event discovery
  -> connection
  -> dynamic prop resolution
  -> create source + canvas
  -> map payload into target flow
  -> capture representative event
  -> select typed sample
  -> test target flow on development
  -> replay captured event against development
  -> publish workflow
  -> optional live replay
```

Skipping capture/sample selection commonly leaves downstream output untyped. Skipping mapping commonly produces a green trigger execution with empty flow inputs.

## IDs

| ID | Used by |
| --- | --- |
| Canvas `triggerId` | `trigger get`, `rename`, `toggle`, `delete` |
| Numeric `backendTriggerId` | `trigger events`, `replay` |
| `triggerNodeId` | `trigger select-event` |
| Captured `eventId` | `trigger event`, `select-event`, `replay --event` |

Always copy these from the transport output.

Development replay is the normal pre-publish check:

```bash
cai trigger replay --trigger <backendTriggerId> --event <eventId> --target dev --json
```

Use `--target live --confirm-live` only for an intentional post-publish live verification.

## Enable/disable

`cai trigger toggle <triggerId>` flips state; it is not a setter. Read `cai trigger list --json` before and after. Do not toggle blindly.

MCP: call `trigger_set_enabled` with `enabled: true` or `enabled: false`.

## Debugging a missing run

1. `cai trigger list --json`: enabled, correct flow, recoverable backend ID.
2. `cai workflow status --json`: published/active live state.
3. `cai trigger events --trigger <backendId>`: did ingestion capture the event?
4. `cai trigger event <eventId>`: does payload match the selected sample?
5. `cai exec list --json`: did execution start?
6. `cai exec tree <executionId> --json`: where did it stop?

No captured event means source/delivery/configuration. Captured event with no execution means enable/live/ingestion routing. An execution with empty inputs means mapping.
