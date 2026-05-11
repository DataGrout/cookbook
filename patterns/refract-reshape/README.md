# Pattern: Refract & Reshape

## What it does

`prism.refract` is the semantic normalization step in most pipelines: it takes free-form text or structured data and reshapes it into a predictable format, entity/attribute facts, a flat table, a typed schema, using a natural language goal. Zero-credit when the result is cached.

## Tools used

- `prism.refract` — semantic transformation (~1 credit)
- `logic.assert` — persist the result (optional, 1 credit)
- `frame.group` / `logic.tabulate` — downstream consumption (0 credits)

## Credits cost

~1 credit per refract call. Subsequent reads from `cache_ref` are free.

## Core pattern

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

raw = """
Q3 Revenue: $4.2M (+18% YoY)
Headcount: 142 (was 97)
Churn rate: 3.1%
NPS score: 67
Top customer segment: Enterprise (64% of ARR)
"""

result = client.prism.refract(
    goal="""Extract business metrics as attribute facts.
    Entity: company_q3
    Attributes: revenue_usd, revenue_growth_pct, headcount, churn_rate_pct, nps_score, top_segment""",
    data=raw
)

# result["result"] is now a list of LC-compatible facts
# [
#   {"type": "attribute", "entity": "company_q3", "attribute": "revenue_usd", "value": 4200000},
#   {"type": "attribute", "entity": "company_q3", "attribute": "revenue_growth_pct", "value": 18},
#   ...
# ]

client.logic.assert_facts(facts=result["result"], namespace="business_metrics")
```

## Reshape modes

### Free-form text -> LC facts

```python
result = client.prism.refract(
    goal="Extract entity and attribute facts. Entity: <name>. Attributes: <list>",
    data=any_text
)
# Returns: list of {"type": ..., "entity": ..., "attribute": ..., "value": ...}
```

### Structured JSON -> flat table rows

```python
result = client.prism.refract(
    goal="Flatten to rows with columns: name, plan, mrr, status",
    data=nested_json_string
)
# Returns: list of dicts with consistent keys
```

### Table -> grouped summary

```python
result = client.prism.refract(
    goal="Group by region, sum revenue, count customers per group",
    data=client.logic.tabulate(query="...", namespace="...")["cache_ref"]
)
```

### Text -> typed schema

```python
result = client.prism.refract(
    goal="""Extract as JSON with schema:
    { "product": str, "price_usd": float, "in_stock": bool, "category": str }
    Return a list of objects, one per product mentioned.""",
    data=catalog_page_text
)
```

## Piping via cache_ref

Refract accepts `cache_ref` from any prior call and returns its own `cache_ref` for downstream tools. No data copying needed.

```python
# Step 1: Search
search = client.inference.search(query="DataGrout Q1 2026 funding news")

# Step 2: Refract the search result (uses cache_ref, not raw text)
facts = client.prism.refract(
    goal="Extract: company name, funding amount, round type, lead investor, date",
    data=search["cache_ref"]  # <- pipe directly
)

# Step 3: Store
client.logic.assert_facts(facts=facts["result"], namespace="funding_events")

# Step 4: Analyze (downstream reads from cache_ref)
analysis = client.prism.analyze(
    goal="Identify funding trends by round type",
    data=facts["cache_ref"]  # <- pipe from refract
)
```

## When to use refract vs analyze

| Situation | Use |
|-----------|-----|
| Normalize data to a consistent structure | `prism.refract` |
| Summarize, interpret, or draw conclusions | `prism.analyze` |
| Extract specific fields from free-form text | `prism.refract` |
| Answer a question about data | `prism.analyze` |
| Prepare data for LC storage | `prism.refract` -> `logic.assert` |
| Prepare data for visualization | `prism.refract` -> `prism.chart` |

## Batch refract

```python
import asyncio

async def refract_all(items: list[str], goal: str) -> list:
    results = []
    for item in items:
        r = client.prism.refract(goal=goal, data=item)
        results.extend(r["result"])
    return results

# Or as a flow.into pipeline
plan = client.flow.into(
    steps=[
        {"tool": "prism.refract", "args": {"goal": "extract facts", "data": "$input"}},
        {"tool": "logic.assert", "args": {"namespace": "batch_facts"}}
    ]
)
```

## Tips

- **Be specific in `goal`**: name the entity, list the attributes, specify types. Vague goals produce inconsistent field names.
- **Use `cache_ref` as input**: avoids re-sending large payloads; refract detects and reads it automatically.
- **Refract before assert**: raw text can't go into LC directly — refract normalizes it to typed facts first.
- **Check `result` length**: if the input has 10 items, you should get ~10× N attribute facts. Spot-check a few.

## See also

- [assert-and-query](../../recipes/logic/assert-and-query/) — store refracted facts in LC
- [research-to-report](../../recipes/combined/research-to-report/) — search -> refract -> assert -> analyze pipeline
- [competitive-intel-pipeline](../../recipes/combined/competitive-intel-pipeline/) — refract used at scale
