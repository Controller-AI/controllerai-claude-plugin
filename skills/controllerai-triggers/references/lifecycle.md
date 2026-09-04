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
| Canvas `triggerId` | `trigger get`, `rename`, `set-enabled`, `delete`, `test-events` |
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

`cai trigger set-enabled <triggerId> --enable|--disable` sets the requested state idempotently. Pass exactly one state flag.

MCP: call `trigger_set_enabled` with `enabled: true` or `enabled: false`.

## Lifecycle flag families

```bash
cai trigger set-enabled <triggerId> --enable|--disable --json
cai trigger test-events <triggerId> --allow [--acknowledge-dual-run] --json
cai trigger test-events <triggerId> --disallow --json
cai trigger replay --trigger <backendTriggerId> --target dev --json
cai trigger replay --trigger <backendTriggerId> --target live --confirm-live --json
cai workflow publish --acknowledge-dual-dispatch --json
```

Use enable/disable for the trigger definition, allow/disallow for ambient development capture, and target for replay rail. `--acknowledge-dual-run` applies only while arming. `--acknowledge-dual-dispatch` applies only when publishing an armed trigger. `--confirm-live` applies only to one Live replay. These acknowledgements are not interchangeable.

The allow flag is sticky even when the trigger is disabled. Always disallow after capture or an abandoned setup so later re-enablement cannot silently rearm development dispatch.

## Debugging a missing run

1. `cai trigger list --json`: enabled, correct flow, recoverable backend ID.
2. `cai workflow status --json`: published/active live state.
3. `cai trigger events --trigger <backendId>`: did ingestion capture the event?
4. `cai trigger event <eventId>`: does payload match the selected sample?
5. `cai exec list --json`: did execution start?
6. `cai exec tree <executionId> --json`: where did it stop?

No captured event means source/delivery/configuration. Captured event with no execution means enable/live/ingestion routing. An execution with empty inputs means mapping.
