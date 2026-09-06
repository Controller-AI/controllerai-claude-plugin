---
name: controllerai-ai-nodes
description: Configure Controller AI AI-powered workflow actions for generation, extraction, classification and structured results, or call a Controller AI agent from a flow. Use when deterministic workflow steps need language-model reasoning.
---

# AI inside a workflow

Use an AI action when a flow needs generation, extraction, classification or summarization. Use ordinary data/condition nodes for exact validation and CRUD. A user-facing conversational assistant is an agent; an AI node is one step in a workflow.

Discover the available AI action and current properties. Configure the connection/model, prompt and output contract using the action's actual schema. For an action calling a Controller AI agent, select the intended agent and verify which conversation/result fields it returns.

## Structured results

Define the schema before downstream mappings. State the required fields, allowed statuses and missing-data behavior in the prompt. Configure the action's structured output mechanism when available. A prompt saying “return JSON” alone does not establish a typed output contract.

For contact research, separate evidence from interpretation. Return source URLs, matched identity, confidence and explicit missing/ambiguous status. “Not found in this search” does not establish that a contact method does not exist. Do not fill an email field with an invented pattern.

A research provider can return variable or unstructured material. Add a typed normalization step when the final flow promises stable fields. Test missing fields, conflicting matches and provider failure as well as the happy path.

## Verify

Run with representative input, inspect the actual output schema and value, then wire downstream nodes from the confirmed output. Verify the final flow return contract. If a connected provider rejects authentication, report the connection and error code without echoing credentials.

Configuration commands: [reference](references/command-reference.md). Action selection and testing: [details](references/selection-and-testing.md).
