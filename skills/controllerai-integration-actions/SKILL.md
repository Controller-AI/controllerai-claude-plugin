---
name: controllerai-integration-actions
description: Discover and attach ready integration actions directly to Controller AI agents, configure connections and dynamic properties, and verify external app operations. Use when one existing action satisfies an agent capability.
---

# Ready actions as agent tools

A ready integration action is one existing operation: search a service, read an object, create a record or send a message. Attach it directly to an agent when that operation is the capability. Use a workflow tool when you also need deterministic validation, multiple ordered steps, transformation, persistence or a reusable typed contract.

## Discover and configure

Search by the intended operation, inspect the matching integration, then read its exact action key and properties. Prefer capability evidence over a similar integration name. Native actions and external provider actions can have different authorization requirements.

Select an existing matching connection and verify its identity. If a new connection is needed, return the connection URL and let the user complete provider authorization. Never request provider credentials in chat. An “active” stored connection still needs a representative action test to prove current authorization.

Attach the action with its exact integration/action/connection and a description that tells the agent when to use it. Inspect configurable inputs and reload dependent options after selector changes. Choose the tool's runtime confirmation setting for the requested experience.

## Verify the attached capability

Read the saved tool list and test in a draft conversation. Inspect the chosen action, arguments, terminal result and intended external effect. A direct action does not create a workflow execution. For a failed or uncertain mutation, inspect external state before repeating it.

Connection rebinding, detachment or revocation can affect Live independently of publishing the agent. If the user requested release, publish the updated agent configuration after testing and verify a Live conversation. Do not publish merely because the draft is ready.

Discovery, attachment, update and removal commands: [reference](references/command-reference.md).
