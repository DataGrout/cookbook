# Recipe: Trend Analysis Pipeline

> **Difficulty:** Intermediate  
> **Credits:** 1–3  
> **Time:** ~5 minutes

## What it does

Chain `data.filter` -> `frame.group` -> `math.trend` -> `prism.analyze` inside a single `flow.into` plan. Zero-credit compute steps reduce the data to a statistical summary; a single LLM call interprets it. The result is a full trend narrative with anomaly flags, delivered for the cost of one analysis call.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `data.filter` | 0 | Drop nulls, out-of-range values |
| `frame.group` | 0 | Aggregate by time bucket and dimension |
| `math.trend` | 0 | Fit trend lines, detect inflections |
| `math.describe` | 0 | Compute summary statistics |
| `prism.analyze` | 1–3 | Narrative synthesis and anomaly flags |

## The Recipe

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# Your time-series data: CSV string, JSON array, or a cache_ref from a prior step
raw = client.perform("data-grout@1/data.load@1", {
    "source": "sales_daily.csv"   # or pass inline JSON
})
data_ref = raw["cache_ref"]

result = client.perform("data-grout@1/flow.into@1", {
    "plan": [
        {
            "tool": "data-grout@1/data.filter@1",
            "output": "clean",
            "args": {
                "cache_ref": data_ref,
                "query": "revenue IS NOT NULL AND revenue > 0 AND date IS NOT NULL"
            }
        },
        {
            "tool": "data-grout@1/frame.group@1",
            "output": "grouped",
            "args": {
                "cache_ref": "$clean",
                "by": ["week", "region"],
                "agg": {"revenue": ["sum", "mean"], "orders": "sum"}
            }
        },
        {
            "tool": "data-grout@1/math.trend@1",
            "output": "trends",
            "args": {
                "cache_ref": "$grouped",
                "column": "revenue_sum",
                "window": 4,            # 4-week rolling average
                "detect_changepoints": True
            }
        },
        {
            "tool": "data-grout@1/math.describe@1",
            "output": "stats",
            "args": {
                "cache_ref": "$grouped",
                "columns": ["revenue_sum", "orders_sum"]
            }
        },
        {
            "tool": "data-grout@1/prism.analyze@1",
            "output": "narrative",
            "args": {
                "goal": "Identify which regions are growing, which are declining, and flag any anomalous weeks. Highlight the top opportunity.",
                "data": "$trends",
                "mode": "causal"
            }
        }
    ],
    "refract": "Key findings in 3 bullet points"
})

print(result["execution_result"]["narrative"])
print(result["execution_result"]["_refract"])   # the 3-bullet summary
```

## How step references work

Each step's `output` name becomes a variable referenceable in later steps as `$name`:

```
$clean    -> filtered rows
$grouped  -> grouped aggregates
$trends   -> trend fit + changepoints
$stats    -> describe statistics
```

Steps execute in order. The data never re-enters the LLM context until `prism.analyze`, `data.filter`, `frame.group`, `math.trend`, and `math.describe` are pure WASM compute.

## Interpreting the output

`math.trend` returns:
- `slope` — positive/negative trend direction per period
- `changepoints` — weeks where the trend changed significantly
- `forecast` — next N periods (if requested)
- `r_squared` — fit quality (0–1)

`prism.analyze` with `mode: "causal"` tries to explain *why* trends shifted, not just *that* they did.

## Variations

**Weekly KPI dashboard:**
```python
# Group by week only, analyze top-level trends
"by": ["week"],
"agg": {"revenue": "sum", "new_customers": "sum", "churn_count": "sum"}
```

**Forecast next quarter:**
```python
{
    "tool": "data-grout@1/math.trend@1",
    "args": {
        "cache_ref": "$grouped",
        "column": "revenue_sum",
        "forecast_periods": 12   # 12 weeks forward
    }
}
```

**Add a conditional alert step:**
```python
# After math.trend, route to different tools based on trend direction
{
    "type": "conditional",
    "input": "trends",
    "branches": [
        {
            "when": [{"field": "slope", "op": "lt", "value": 0}],
            "label": "declining",
            "then": {"tool": "data-grout@1/prism.analyze@1", "args": {
                "goal": "Diagnose the revenue decline and recommend corrective actions",
                "data": "$trends"
            }}
        }
    ],
    "else": {"tool": "data-grout@1/prism.analyze@1", "args": {
        "goal": "Identify growth drivers and expansion opportunities",
        "data": "$trends"
    }}
}
```

**Save as a reusable skill:**
```python
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [...],
    "save_as_skill": True,
    "skill_name": "weekly-trend-report",
    "input_schema": {"source": {"type": "string"}, "regions": {"type": "array"}}
})
skill_id = result["skill_id"]

# Run it later with new data:
client.perform(skill_id, {"source": "sales_q2.csv"})
```

## See also

- [zero-credit-pipeline](../../../patterns/zero-credit-pipeline/) — the underlying pattern this recipe uses
- [conditional-routing](../conditional-routing/) — branching on trend results
- [log-processing](../log-processing/) — another flow.into + flow.route pipeline
