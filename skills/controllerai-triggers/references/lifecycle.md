<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Trigger lifecycle

## State sequence

```text
source decision (basis)
  -> exact node and event
  -> connection
  -> prop and option resolution (latest dynamicPropsId)
  -> create source and canvas trigger (triggerId, backendTriggerId, triggerNodeId, runSubflowNodeId)
  -> declare the flow inputs
  -> capture a representative event, select it
  -> bind one mapping prop per input on the run-subflow node
  -> run the target flow on development
  -> replay the captured event on development
  -> audit every trigger, then authorized publish
  -> optional live replay
```

Skipping capture or selection leaves the trigger output untyped. Skipping the mapping produces a green trigger execution with empty flow inputs. Leaving ambient dispatch allowed can produce double runs after publish.

## IDs

| ID | Used by |
| --- | --- |
| Canvas `triggerId` | get, rename, set-enabled, delete, test-events |
| Numeric `backendTriggerId` | captured events, replay (`--trigger`) |
| `triggerNodeId` | select-event |
| `runSubflowNodeId` | the mapping props, one per flow input |
| Captured `eventId` | event read, select-event, replay |

Copy them from the responses. A replaced trigger has new IDs and possibly a new webhook URL.

## Switches and acknowledgements

| Switch | Flags | Meaning |
| --- | --- | --- |
| Definition state | `set-enabled --enable` / `--disable` | Whether the trigger starts its flow; development edit until published |
| Ambient development dispatch | `test-events --allow --acknowledge-dual-run` / `--disallow` | Real events also execute the development canvas; sticky permission bit, survives disable |
| Replay rail | `replay --target dev` / `--target live --confirm-live` | Which version one captured event executes |
| Publish while armed | `workflow publish --acknowledge-dual-dispatch` | User's informed acceptance of double runs and double billing |

Each acknowledgement belongs to exactly one operation. Arming, publishing armed and live replay are three separate user decisions.

## Lifecycle fields

The list and get reads report `devCanvasEnabled` at the top level and, under `deployment`, `liveCanvasEnabled` (null when no live canvas exists: never published, or this trigger was added after the last publish), `inLiveVersion`, `devDispatchArmed` (enabled and allowed) and `dualDispatchLive`; get adds captured event counts and issues. The audit summarizes coverage and double-run risk before a publish and names the disarm step for each armed trigger.

## Missing-run diagnosis

1. The trigger list: enabled, correct flow, recoverable backend ID?
2. The workflow status: published, and is the live version active?
3. The captured events for the backend trigger ID: did ingestion capture the event?
4. The event read: does the payload match the selected sample?
5. The execution list: did an execution start?
6. The execution tree: where did it stop?

No captured event points to the source, delivery or configuration. A captured event with no execution points to enablement, publish state or ingestion routing. An execution with empty inputs points to the mapping. Development works but live does not: compare the enabled state and the published configuration, and remember that live runs read the live data store.
