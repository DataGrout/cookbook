# Recipe: First Flow

## What it does

Compose two tools with `flow.into`. Demonstrates the plan structure, step references (`$step_id.result`), and the `analyze` post-processor.

## Tools used

- `flow.into` — sequential tool composition
- `prism.refract` — semantic reshape
- `prism.analyze` — LLM synthesis

## Credits cost

2–4 credits (refract: 1, analyze: 1–3).

## The Recipe

A flow is a plan, a list of steps where each step can reference previous results with `$step_id.result`.

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Raw data to process
raw_data = """
Q1 Sales by Region:
North: $1.2M (up 12%)
South: $890K (down 3%)
East: $1.5M (up 28%)
West: $670K (flat)
"""

result = client.flow.into(
    steps=[
        {
            "id": "reshape",
            "tool": "prism.refract",
            "args": {
                "goal": "Extract region, revenue, and growth_pct as structured records",
                "data": raw_data
            }
        },
        {
            "id": "analyze",
            "tool": "prism.analyze",
            "args": {
                "goal": "Which regions need attention and why?",
                "data": "$reshape.result",  # Reference to step 1 output
                "mode": "deductive"
            }
        }
    ]
)

print(result["steps"]["reshape"]["result"])  # Structured records
print(result["steps"]["analyze"]["result"])  # Analysis
```

### Understanding `$step_id.result`

Step references are resolved at runtime:
- `$reshape.result` — the full output of the `reshape` step
- `$reshape.result.records` — a field within that output
- Steps execute in order; forward references are not allowed

### Flow with analyze post-processor

```python
# Shorthand: run a flow and analyze the final output in one call
result = client.flow.into(
    steps=[...],
    analyze={
        "goal": "Summarize the findings",
        "mode": "exploratory"
    }
)
print(result["analysis"])
```

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

const rawData = `
Q1 Sales by Region:
North: $1.2M (up 12%)
South: $890K (down 3%)
East: $1.5M (up 28%)
West: $670K (flat)
`;

const result = await client.perform('data-grout@1/flow.into@1', {
    steps: [
        {
            id: 'reshape',
            tool: 'prism.refract',
            args: {
                goal: 'Extract region, revenue, and growth_pct as structured records',
                data: rawData
            }
        },
        {
            id: 'analyze',
            tool: 'prism.analyze',
            args: {
                goal: 'Which regions need attention and why?',
                data: '$reshape.result',
                mode: 'deductive'
            }
        }
    ]
});

console.log(result.steps.reshape.result);
console.log(result.steps.analyze.result);
```

## Try it

```python
# Paste the raw_data example above and run
python first_flow.py
```

## Variations

**Parallel steps**, steps with no dependencies can run concurrently:
```python
{"id": "a", "tool": "prism.refract", "args": {...}},
{"id": "b", "tool": "prism.refract", "args": {...}},  # runs in parallel with "a"
{"id": "c", "tool": "prism.analyze", "args": {"data": "$a.result, $b.result"}}
```

**Save as skill**, reuse the flow later:
```python
result = client.flow.into(steps=[...], save_as_skill=True, skill_name="q1-analysis")
# Later:
result = client.perform(skill_id=result["skill_id"], args={"data": new_data})
```

## Next steps

- [first-logic-cell](../first-logic-cell/) — store flow results as persistent facts
- [zero-credit-pipeline](../../patterns/zero-credit-pipeline/) — flows with zero-credit transforms
