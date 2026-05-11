# Pattern: Cache-Ref Composition

## What it does

Pipe results between tools via `cache_ref`. Show how tool A produces output -> `cache_ref` returned -> tool B consumes `cache_ref` -> chain continues. Avoids moving large datasets through the LLM context window.

## Tools used

- Any tool that returns a `cache_ref`
- `ephemerals.list` — inspect active cached datasets
- `ephemerals.get` — retrieve cached data

## Credits cost

Varies by tools used. Cache operations themselves are free.

## Understanding cache_ref

Most DG tools that produce data return two things:
1. A human-readable summary (shown in the LLM context)
2. A `cache_ref`, a pointer to the full dataset stored server-side

```python
result = client.data.filter(input=data, query="amount > 0")
print(result["summary"])    # "Filtered to 847 rows (153 removed)"
print(result["cache_ref"])  # "rc_AbCdEf1234"
print(result["data"])       # None, full data is server-side, not in context
```

The `cache_ref` is ephemeral (default: 1 hour) and can be passed directly to the next tool's `input` parameter.

## The Recipe

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Step 1: Load -> cache_ref
step1 = client.data.load(source="large-dataset.csv")
ref1 = step1["cache_ref"]
print(f"Loaded: {step1['summary']}")  # "1,000 rows, 8 columns"

# Step 2: Filter -> new cache_ref
step2 = client.data.filter(input=ref1, query="revenue > 10000")
ref2 = step2["cache_ref"]
print(f"Filtered: {step2['summary']}")  # "412 rows remaining"

# Step 3: Group -> new cache_ref
step3 = client.frame.group(
    input=ref2,
    by="product_category",
    agg={"revenue": "sum", "count": "count"}
)
ref3 = step3["cache_ref"]

# Step 4: Analyze (consumes cache_ref, returns text in context)
analysis = client.prism.analyze(
    goal="Which product categories drive the most revenue?",
    data=ref3  # cache_ref passed directly
)
print(analysis["result"])  # Full analysis text
```

## Inspecting the cache

```python
# List all active cache entries
entries = client.ephemerals.list()
for entry in entries["entries"]:
    print(f"{entry['ref']}: {entry['rows']} rows, expires {entry['expires_at']}")

# Retrieve cached data for inspection
data = client.ephemerals.get(ref="rc_AbCdEf1234")
print(data["preview"])  # First 10 rows as table
```

## Cache lifetime

| Context | Default TTL |
|---------|-------------|
| Interactive session | 1 hour |
| Background task | 4 hours |
| Explicitly extended | Up to 24 hours |

```python
# Extend a cache entry
client.ephemerals.extend(ref=ref1, ttl_hours=4)
```

## Cross-tool composition

cache_refs work across any tools that accept `input`:

```python
# Research result -> logic facts
search_result = client.inference.search(query="...")
facts_ref = search_result["cache_ref"]

normalized = client.prism.refract(
    goal="Extract entity facts",
    data=facts_ref
)
client.logic.assert_facts(facts=normalized["cache_ref"], namespace="research")
```

## Variations

**Branching:** same ref consumed by multiple steps

```python
ref = step1["cache_ref"]
stats = client.math.describe(input=ref)     # consume from original
trends = client.math.trend(input=ref)       # consume from same original
# ref is read-only; multiple consumers are fine
```

**Materializing:** pull data into local context when needed

```python
local_data = client.ephemerals.get(ref=ref3)["data"]
# Now it's in your local Python dict: useful for small results
```

## See also

- [zero-credit-pipeline](../zero-credit-pipeline/) — uses cache_refs throughout
- [background-tasks](../background-tasks/) — cache_refs survive into background tasks
