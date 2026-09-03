# Data rails and safety

## What publishing copies

| Asset | Workflow publish |
| --- | --- |
| Graph, props, expressions, schemas | Promoted to live |
| Development records | Not copied |
| Existing live records | Remain on live |

Use `cai data copy-to-live --type <custom.type>` only for an intentional data promotion.

MCP: `data_copy_to_live` requires the explicit acknowledgement `env: live`.

## Safe schema update

1. `cai schema get <type> --json`
2. Search workflow expressions and node props that reference its fields.
3. Prepare the complete replacement field list.
4. Identify dropped/renamed/type-changed fields and their record impact.
5. Run `cai schema update`.
6. Read schema, collections, and affected expression contexts again.
7. Rerun stateful flows.

Field IDs are contracts. Prefer keeping an ID stable and changing only its display label.

## Destructive operations

- `data delete` soft-deletes one record.
- `data delete-many --all --yes` is broad and REST-backed on either rail. With `--env live`, the deletion immediately changes what the published workflow reads.
- `schema delete` removes access to the whole collection and breaks its type references.
- CSV `--mode commit` appends rows on the selected rail; a live commit is immediately visible to the published workflow. `--mode dryRun` writes nothing on either rail.
- `copy-to-live` changes production data.

Resolve exact workflow, environment, type, and record targets before these actions. Do not use a broad deletion to repair a schema mistake.

## Dev/live diagnosis

If a manual test passes but a trigger does not:

```bash
cai data collections --env dev --json
cai data collections --env live --json
cai data query --type <type> --env dev --json
cai data query --type <type> --env live --json
```

Different row counts are expected until data is intentionally copied or written on both rails.
