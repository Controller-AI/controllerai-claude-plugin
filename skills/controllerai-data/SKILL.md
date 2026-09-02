---
name: controllerai-data
description: Define Controller AI custom data types and safely read, write, import, copy, and debug workflow records across development and live stores. Use when a workflow needs durable state, custom structured types, collections, record queries, CSV import, workflowData expressions, or data migration between dev and live.
---

# Controller AI data

The `cai ...` examples are canonical grammar. Use the transport selected in `controllerai-start` and its MCP tool map; never mix transports in one task.

Controller AI has two separate concerns:

- A `custom.*` schema defines fields and makes a collection available.
- Development and live record stores hold separate rows. Publishing graph/configuration does not copy records.

## A type vs a stored record

`custom.customer` and `dataRecord.custom.customer` are different things, and the difference carries through everything:

- `custom.customer` is the TYPE — the shape, like a TypeScript interface. A value of this type is just data in that shape.
- `dataRecord.custom.customer` is an ACTUAL STORED RECORD of that shape — the same fields plus record identity (`id`, `createdAt`, `updatedAt`) proving it lives in the collection.

In the product UI this appears as the "data record" checkbox when selecting a type for a flow input or run-subflow-on-list: unchecked means "data shaped like a customer", checked means "a real stored customer record". Users find this checkbox confusing — be ready to explain it in these terms.

Consequences:

- Only real records satisfy a `dataRecord.*` input. An expression that CONSTRUCTS record-shaped objects (e.g. a `.map()` that builds new objects) produces `custom.*`-shaped values with no identity — pass fetched records through instead.
- Record metadata (`id`, `createdAt`) exists only on `dataRecord.*` values; on plain `custom.*` values those paths resolve empty.

## Data nodes: update semantics

- "Update multiple data records" takes an expression typed `list.dataRecord.custom.<type>` — real stored records passed through from a query or `workflowData(...)`. The batch is ONE transaction: if any resolved record is stale (deleted since fetched) or foreign to the collection, the node errors and NOTHING updates. A failed run therefore means zero changes — fix and rerun safely.
- On the single-record "Update data" node, every ENABLED field is written. Enable only the fields you intend to change; disabled fields keep their stored values.

## Define a schema deliberately

Inspect existing types first:

```bash
cai schema list --json
cai schema get custom.<name> --json
```

Create a type with stable field IDs:

```bash
cai schema create custom.customer \
  --display "Customer" \
  --field email:text:"Email" \
  --field score:number:"Score" \
  --json
```

Creating the schema creates/synchronizes its collection. `cai data collection create` is only an alias for schema creation.

MCP: `cai data collection create` is merged into `schema_type_create`.

`schema update --field` replaces the complete field list; it does not append — this is deliberate, so a subset CAN intentionally replace the whole schema. Read the current schema, preserve intended fields, and disclose any removals. The transport response warns about every field being dropped and echoes back the COMPLETE stored shape after the write — always read that echo and confirm it matches intent. Deleting a schema hides its collection and records and breaks dependent expressions — tell the user, and know it is recoverable (see below).

## How schema changes affect stored records

Schema changes never hard-delete data. There is no delete path for records in schema sync — everything is deactivation and hiding:

- Delete a type → its collection goes INACTIVE on the next saved state; every record row survives. Re-add the same `custom.*` key and the same collection reactivates with all records intact.
- Delete a field → its values stay inside each record, unaddressable. Re-add the SAME field id and the old values reappear.
- Add a field → purely additive; existing records read the new field as null.
- Change a field's type → values are never converted or erased; incompatible values read as null and return if the type is changed back. The schema hash changes, which blocks copy-to-live until dev and live match again.
- Publish applies the same deactivate/reactivate logic to live. Live records are hidden, never deleted.
- Restoring an older workflow state rolls back the DEFINITION (graph and schemas), never the data. The record store is not versioned and only moves forward: a deleted type reactivates with whatever the store holds NOW — records created, edited, or soft-deleted after the restored state stay exactly as they are. Restore is not a data snapshot.

The truly destructive operations are record-level only: `data delete`, `data delete-many`, and overwriting edits. Those have no undo — treat them with explicit user intent; everything schema-shaped is safely reversible.

One rename trap: renaming a type key is create-new + deactivate-old — all existing records stay stranded under the OLD key's collection, invisible. Migrate records explicitly (export/import) or rename back to recover.

## Respect field-key asymmetry

```bash
cai data fields --type custom.customer --json
```

- `data create` and `data update` take field IDs.
- Query/get results are keyed by field display names.
- CSV columns are also matched by display names.

Never feed a queried record directly into a write. Build a new fields object keyed by the `writeKey` mapping and omit reserved `id`, `createdAt`, and `updatedAt`.

Fields may be typed `date`. Their values are strict ISO-8601 strings (`"2026-06-05"` or `"2026-06-05T12:30:00Z"`) on both write and read; anything else is rejected. Reserved `createdAt`/`updatedAt` are currently `text` — use `.toDate()` before date operations on them.

## Work on the correct rail

Development is the default:

```bash
cai data collections --env dev --json
cai data query --type custom.customer --env dev --json
cai data create --type custom.customer --fields '{"email":"a@example.com","score":5}' --env dev --json
```

MCP: every data write requires explicit `env`; reads default to `dev` when `env` is omitted.

Use `--env live` only when the user intends to inspect or mutate production state. Before a live write, say that published/triggered runs read this store.

`data query --query` is plain-text search over serialized records and metadata, not a field predicate. Perform exact field filtering inside workflow expressions or inspect results explicitly.

## Delete records explicitly

Record deletion is available only through the control surface; no workflow data node exposes it. Delete one known record or an explicit set of IDs:

```bash
cai data delete <recordId> --type custom.customer --env dev --json
cai data delete-many --type custom.customer --records <id1,id2> --env dev --json
```

Delete every record in a collection only with explicit user intent; `--all` also requires `--yes`:

```bash
cai data delete-many --type custom.customer --all --yes --env dev --json
```

MCP: `data_record_delete_many` requires `env: dev` and exactly one of `recordIds` or `all: true`; it has no `yes` parameter.

The default rail is development. Bulk deletion is development-only. Use `--env live` only for an intentional single-record production deletion, then query that same rail to verify the result.

## Import and promote

Validate CSV before writing:

```bash
cai data import --type custom.customer --file customers.csv --mode dryRun --json
cai data import --type custom.customer --file customers.csv --mode commit --json
```

MCP: pass CSV text inline as `content` and `env: dev` to `data_record_import`; live import is unavailable.

The commit appends. Verify result counts and query the imported rows.

To seed live from development:

```bash
cai workflow publish --json
cai data copy-to-live --type custom.customer --json
cai data collections --env live --json
```

MCP: call `data_copy_to_live` with the explicit acknowledgement `env: live`.

The live schema must exist through a publish before copying. Treat copy-to-live as a production mutation and report counts/results.

## Workflow expressions

`workflowData("custom.customer")` requires an active collection on the executing rail. Read `controllerai-expressions` for filtering. Test stateful behavior against development, then independently verify required live seed data.

Read [references/data-rails.md](references/data-rails.md) before schema rewrites, deletion, bulk deletion, import, or copy-to-live.
