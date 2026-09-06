---
name: controllerai-triggers
description: Build and verify Controller AI event-driven workflows using schedules, webhooks, email and integration events. Use for trigger discovery, configuration, samples, event mapping, replay and enabling live execution.
---

# Run workflows from events

A trigger captures a schedule, webhook, email or integration event and maps it into a flow's typed inputs. It lets the same reusable processing run without a chat message. The trigger configuration, captured samples and execution state are separate concerns.

## Build the event path

1. Discover the trigger component and inspect its properties, authorization and output schema.
2. Create/select the destination flow and define the inputs it needs.
3. Configure the trigger using the exact component identity. Choose selectors before dependent properties and bind any required connection.
4. Capture a representative event. Use the returned webhook URL, trigger email address or provider action; do not invent an endpoint.
5. Inspect/select the captured sample and map its fields into flow inputs using the actual expression context.
6. Replay the sample on development and verify received inputs, flow outputs and expected record or external effects.
7. When live operation is authorized, publish the workflow, enable the trigger as needed and verify an actual live event.

Use `trigger_set_enabled` for a reversible pause/resume and `trigger_delete` to remove a retired trigger. Capture can exist before publication; publication controls the live executable definition. Replaying an event can perform real external actions, so use the existing authorization and intended test scope. A recorded event is not proof the flow executed.

## Diagnose by stage

If there is no event, inspect trigger identity, connection, registration and the source action. If the event exists but inputs are wrong, inspect selected sample and mappings. If the flow starts but fails, follow the execution ID with `controllerai-runs`. Inspect enabled state and published configuration when development works but live does not.

Schedules need an explicit timezone and intended cadence. Follow the user's cadence; do not infer a production schedule from a test sample. Avoid duplicate triggers while debugging: keep the original trigger and event IDs.

Commands: [reference](references/command-reference.md). Capture, replay and lifecycle details: [lifecycle](references/lifecycle.md).
