# AI node selection and testing

## Choose the surface

| Need | Surface |
| --- | --- |
| Extract/classify/summarize one payload | Ask AI node |
| Deterministic transformation | Expression or normal action node; avoid AI |
| Conversational assistant with several capabilities | Controller AI agent with workflow tools |
| Workflow starts or messages an existing platform agent | Exact `registry_agent` action |

Do not use AI for logic that can be expressed deterministically and tested exactly.

## Model selection

After provider selection reloads props:

```bash
cai node options <nodeId> --prop <modelProp> --query '<preference>' --json
```

Set only a returned value, and pass its matching displayed label with `cai node set ... --value '<value>' --label '<label>'`. Balance:

MCP: use `node_prop_set` for the canonical `cai node set` write.

- Quality/reasoning complexity
- Latency
- Runtime cost
- Structured-output reliability
- User-stated provider or privacy constraints

If no option meets an explicit requirement, report the gap rather than silently choosing another provider.

## Minimal evaluation set

For a classifier/extractor, test:

1. Clear positive/normal case
2. Clear negative or alternate category
3. Missing or insufficient data
4. Conflicting/ambiguous evidence
5. Malicious or instruction-like text inside the data when applicable

Assert the structured fields and allowed values. A fluent answer with the wrong schema is a failure.

## Runtime evidence

`resolvedConfigs` proves what prompt and settings reached the action. The node output proves what the provider returned. Inspect both. Redact secrets and avoid placing credentials or unnecessary personal data in prompts.
