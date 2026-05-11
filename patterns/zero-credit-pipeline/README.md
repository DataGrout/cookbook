# Pattern: Zero-Credit Pipeline

## What it does

Chain `data.filter` -> `frame.group` -> `math.describe` -> `math.trend` entirely at zero credits. The data never enters the LLM context window. Conclude with a single `prism.analyze` call as the only credit-consuming step.

## Tools used

| Tool | Credits | Tokens | Purpose |
|------|---------|--------|---------|
| `data.filter` | 0 | 0 | Remove invalid rows |
| `frame.group` | 0 | 0 | Aggregate by dimension |
| `math.describe` | 0 | 0 | Compute statistics |
| `math.trend` | 0 | 0 | Detect trends |
| `prism.analyze` | 1–3 | ~500 | Final synthesis |

## Credits cost

1–3 credits total (only the final analysis step).

## The Recipe

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Load your data (CSV, JSON, or cache_ref from a previous step)
data_ref = client.data.load(source="sales-q1.csv")["cache_ref"]

# Zero-credit pipeline
filtered = client.data.filter(
    input=data_ref,
    query="amount > 0 AND region IS NOT NULL"
)["cache_ref"]

grouped = client.frame.group(
    input=filtered,
    by=["region", "product_category"],
    agg={"amount": "sum", "count": "count", "amount": "mean"}
)["cache_ref"]

stats = client.math.describe(
    input=grouped,
    columns=["amount_sum", "amount_mean"]
)["cache_ref"]

trends = client.math.trend(
    input=grouped,
    column="amount_sum",
    window=7  # 7-period moving average
)["cache_ref"]

# Only this step uses credits
analysis = client.prism.analyze(
    goal="Identify regional performance patterns, outliers, and growth opportunities",
    data=trends,  # or stats — pick the richest summary
    mode="exploratory"
)

print(analysis["result"])
```

## Why Zero Credits?

`data.filter`, `frame.group`, `math.describe`, and `math.trend` are pure compute operations implemented as WASM modules in the DG runtime. No LLM inference is involved. They are metered for data volume only.

The pattern: **do all data manipulation at zero credits, then hand a compact statistical summary to a single LLM call.**

The LLM sees a 500-row statistical summary instead of a 1,000-row raw dataset. The analysis is better (structured input) and the cost is a fraction.

## As a flow.into plan

```python
result = client.flow.into(steps=[
    {
        "id": "filter",
        "tool": "data.filter",
        "args": {"input": "$data", "query": "amount > 0"}
    },
    {
        "id": "group",
        "tool": "frame.group",
        "args": {"input": "$filter.result", "by": "region", "agg": {"amount": "sum"}}
    },
    {
        "id": "describe",
        "tool": "math.describe",
        "args": {"input": "$group.result"}
    },
    {
        "id": "trend",
        "tool": "math.trend",
        "args": {"input": "$group.result", "column": "amount_sum"}
    },
    {
        "id": "analyze",
        "tool": "prism.analyze",
        "args": {
            "goal": "Identify patterns and opportunities",
            "data": "$describe.result"
        }
    }
], inputs={"data": data_ref})
```

## Token cost

Steps 1–4 run at zero tokens. Only the final `prism.analyze` call touches the LLM (~300 in, ~400 out). Total: ~700 tokens for a full data analysis pipeline, versus ~35,000 tokens if you passed 1,000 rows directly to the LLM.

If you're tracking with [Lumen](https://github.com/DataGrout/lumen), use `lumen.laps` to confirm the zero-token steps in the lap breakdown.

## Variations

**Pivot instead of group:**
```python
client.frame.pivot(input=filtered, rows="region", columns="quarter", values="amount", agg="sum")
```

**Rolling window analysis:**
```python
client.math.window(input=grouped, column="amount_sum", window=30, fn="rolling_mean")
```

**Outlier detection:**
```python
client.math.describe(input=grouped, outlier_method="iqr", outlier_threshold=1.5)
```

## See also

- [benchmarks/04-data-pipeline](https://github.com/DataGrout/benchmarks) — benchmark using this pattern
- [cache-ref-composition](../cache-ref-composition/) — passing results between tools
