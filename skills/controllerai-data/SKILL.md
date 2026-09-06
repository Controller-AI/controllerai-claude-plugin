---
name: controllerai-data
description: Build durable application state in Controller AI workflow data stores: schemas, record identity, CRUD flows, search, partial updates, relationships, imports and development/live data. Use for CRM backends, trackers, data APIs and persistence.
---

# Workflow data as an application backend

Each workflow can own custom schemas and durable collections. Its flows share those collections, so an agent can use a set of flows as a CRM or application API. Creating a schema makes the collection available; creating a flow output only returns a value and does not store it.

`custom.contact` describes a shape. `dataRecord.custom.contact` is a stored record of that shape, including `id`, `createdAt` and `updatedAt`. Lists of fetched records preserve identity. Use real fetched records as inputs to native Get/Update/Delete actions; an object that merely looks like a contact does not prove record identity.

## What is available

- Native Create, Get, Update, Update Multiple and Delete data-record actions.
- `workflowData("custom.contact")` expressions for query/filter/sort/count and selecting records.
- Typed record relationships, separate dev/live records, schema changes, CSV import validation/commit, and explicit copy-to-live.
- MCP data tools for inspecting, creating, querying, patching and deleting records during setup or debugging. Native workflow nodes provide the reusable capability an agent calls later.

Discover the current `registry_data` action keys and schemas. Do not infer unsupported database guarantees: unique business keys, permissions, relationship deletion policy and concurrent business rules need deliberate application logic or verified platform support.

## Recipe: CRM flows for an agent

1. Create schemas with stable field IDs: companies, contacts, deals and activities. Use fields for business data, record references for relationships where appropriate, and a clear status/archive policy.
2. Define create/get/update/delete/search flow contracts per entity. Include useful descriptions and explicit results for invalid inputs, missing records and duplicate business keys. Do not make an empty record look like success.
3. Create validates input and referenced records before writing. Get selects by record ID. Update fetches the record and patches only requested fields. Delete applies the chosen relationship policy. Search returns matching records and count/page information.
4. Add useful composite reads such as “contact plus company, deals and activity history.” Prefer explicit projections over large unbounded record graphs.
5. Test create → get → search → update → get → delete → missing get. Also test duplicate keys, invalid references, omitted filters, empty matches, zero/false values, list membership, clearing a field and page boundaries.
6. Attach each tested flow as an agent tool when a conversational CRM agent is requested.

## Search expressions

Use current `expr_context` bindings. Typical sources and operations:

```javascript
workflowData("custom.contact").filter(r => r.id === flow.recordId).first()
workflowData("custom.contact").filter(r => r.owner === flow.owner, { ignoreEmptyConstraints: true })
workflowData("custom.contact").filter(r => r.tags.includes(flow.tag))
workflowData("custom.contact").skip(flow.offset).take(flow.limit)
```

`ignoreEmptyConstraints` skips absent values, empty text and empty lists. `false` and `0` remain actual filter values. Without the option, an empty value remains a real comparison. List `.includes(value)` takes an **element**, not a list of elements. A text `.includes` is case-sensitive; for case-insensitive search, store normalized search text and normalize the query before comparing. Test the exact filter against representative rows.

Use `.take(n)` for a count and `.skip(n)` for an offset. Legacy `.itemsFrom(n)`/`.itemsUntil(n)` use one-based boundaries: `.itemsUntil(51)` returns the first 50. Keep page counts and limits explicit; do not label a truncated list as a complete export.

## Partial updates

Single Update writes every enabled field by default. Enable only intended fields for a fixed patch. For a dynamic API patch, enable the optional `$fieldMask` prop, bind it to a `list.text` of schema field IDs, and enable/configure the candidate fields. Only selected fields are written in one update. An empty mask selects none; an unknown or disabled selected field is an error. Omit the mask to keep the existing enabled-field behavior.

This supports list/date/scalar updates without one node per field. Selected `false`, `0`, empty text and empty lists are real updates; omitted mask entries preserve the stored values. Distinguish clearing from omission in the flow contract. Update Multiple applies a patch transactionally to the selected real records; verify its failure behavior for stale or foreign IDs.

## Keys, schemas and environments

Inspect `data_field_list` for write keys. Data-tool writes use stable field IDs; query results can use display labels. Construct writes from the mapping instead of feeding a returned display-keyed object back into a write. Renaming a label is different from changing the stable schema/field key.

Schema field updates replace the full declared field list. Read the existing schema and include every field intended to remain. Removing a schema or field hides its stored data; it does not snapshot or erase the underlying records. Restoring workflow history restores the definition only.

Development is the default test store; MCP writes require explicit `env`. Publishing the workflow does not copy records to Live. Use explicit validated copy/import operations when data promotion is actually intended. Verify state in the same environment as the flow execution.

Detailed schema, import and data commands: [command reference](references/command-reference.md). Environment and reference semantics: [data rails](references/data-rails.md).
