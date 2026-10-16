<!-- generated from .claude/skills by cli/scripts/render-skills.mjs; edit the source -->
# Data rails and safety

## What publishing copies

| Asset | Workflow publish |
| --- | --- |
| Graph, props, expressions, schemas | Promoted to live; live collections are created or synced |
| Development records | Never copied |
| Existing live records | Untouched |

Records reach live only through an explicit copy-to-live, a live import, a live write, or the published workflow's own runs.

## Safe schema change

1. Read the current schema and note every field ID and type.
2. Search the workflow's expressions and node props for references to the fields you will change.
3. Prepare the complete replacement field list; the update replaces the list.
4. Name the fields that will be removed, renamed or retyped, and what their stored values will do: hidden until the same ID returns, null until the same type returns.
5. Update, then read the persisted echo and compare it with the intent.
6. Re-read the expression contexts that referenced the schema and rerun the stateful flows.

Prefer a display change over an ID change. A new field ID is a new field: the old values stay under the old ID.

## Destructive operations

- A record delete is a soft delete with no undo.
- A bulk delete of every record removes the whole collection's rows on the chosen store; on live the published workflow sees it immediately.
- An update overwrites the stored value; on the single Update node every enabled field that resolves to a value is written, including one that resolves to empty text or null.
- A CSV commit appends the valid rows and skips the invalid ones; re-running a commit appends again.
- Deleting a type deactivates its collection (re-adding the key reactivates it) and removes the fields on other types that referenced it; re-adding the type does not restore those fields, restoring an earlier workflow state does.
- Copy-to-live upserts development rows into production by record key: live-only rows stay, matching keys are overwritten, deleted matching keys are restored.

Resolve the exact workflow, store, type and records before any of these. A schema mistake is fixed by restoring the key or the field ID, never by deleting data.

## Development versus live diagnosis

If a manual run passes but a trigger or agent tool does not, compare the two stores:

- `data_collection_list` — `env`, `workflowId`
- `data_record_query` — `env`, `limit`, `offset`, `query`, `type`, `workflowId`

No live collections usually means no publish since a store-eligible schema existed. An empty live collection with a populated development one usually means the data was never promoted or was cleared since. Confirm with publish status, the live schema and record history; different counts are normal until data is intentionally written on both stores.
