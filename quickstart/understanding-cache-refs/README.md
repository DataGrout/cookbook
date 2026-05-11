# Quickstart: Understanding cache_ref

## What you'll learn

Every DG tool that returns data gives you a `cache_ref`, a short pointer to the result stored server-side in the ephemerals layer. Passing `cache_ref` between tools is free, instant, and avoids re-sending large payloads. This walkthrough shows how it works end-to-end.

## Prerequisites

```bash
pip install datagrout-conduit
export DATAGROUT_API_KEY=your-key
```

## Step 1: Call a tool, get a cache_ref back

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

result = client.inference.search(
    query="What is DataGrout and what problem does it solve?",
    quality="standard"
)

print(result["result"])      # the actual answer text
print(result["cache_ref"])   # something like "cr_a1b2c3d4e5f6..."
print(result["expires_at"])  # when this ephemeral expires
```

The `cache_ref` is a handle to the result. The data lives on DG's servers, you hold the key.

## Step 2: Pipe it to another tool for free

Pass `cache_ref` anywhere a tool accepts a `data` argument. DG reads it server-side, no round-trip, no re-sending the text, no credit cost for the data transfer.

```python
# Analyze the search result: 1-2 credits for the analysis, 0 for the data
analysis = client.prism.analyze(
    goal="Extract: product name, core problem solved, primary user, key differentiators",
    data=result["cache_ref"],   # <- the pointer, not the text
    mode="deductive"
)

print(analysis["result"])      # synthesis extracted from the search text
print(analysis["cache_ref"])   # a new cache_ref for the analysis output
```

Each tool produces its own `cache_ref`. You can chain as many as you want.

> **Note:** `prism.refract` reshapes structured data (JSON/records). For extracting structure from plain text, like a search result, use `prism.analyze` instead.

## Step 3: Chain a third tool

```python
# Further analyze: still using cache_refs throughout
summary = client.prism.analyze(
    goal="In two sentences: what makes this product distinctive and who is it for?",
    data=analysis["cache_ref"],   # <- output of previous analyze step
    mode="exploratory"
)

print(summary["result"])
```

You've now run three tools. The data moved between them entirely server-side. You only paid for the tool calls themselves.

## Step 4: Inspect the ephemerals layer directly

You can look up any cache_ref you hold:

```python
# Check what's stored and when it expires
ephemeral = client.ephemerals.get(ref=result["cache_ref"])

print(ephemeral["size_bytes"])   # how large the stored data is
print(ephemeral["expires_at"])   # TTL
print(ephemeral["tool"])         # which tool created it
```

List everything currently stored in your session:

```python
all_refs = client.ephemerals.list()
for e in all_refs["ephemerals"]:
    print(f"{e['ref']}  {e['tool']}  {e['size_bytes']} bytes  expires {e['expires_at']}")
```

## Step 5: Extend a ref you want to keep

By default ephemerals have a session-scoped TTL. If you want to keep a result for a longer pipeline or a follow-up session, extend it:

```python
client.ephemerals.extend(
    ref=facts["cache_ref"],
    ttl_seconds=3600   # keep for another hour
)
```

## Full picture

Here's the complete flow with all the cache_refs made visible:

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Step 1: search: 1 credit
search = client.inference.search(query="DataGrout Logic Cell use cases", quality="standard")
print(f"search -> {search['cache_ref']}")

# Step 2: analyze the text result: 1-2 credits, data free
use_cases = client.prism.analyze(
    goal="List the top 5 use cases with a one-sentence description each",
    data=search["cache_ref"],
    mode="exploratory"
)
print(f"analyze -> {use_cases['cache_ref']}")

# Step 3: store the summary in a logic cell: 1 credit, data free
client.logic.assert_facts(
    facts=[{"type": "context", "key": "use_case_summary", "value": use_cases["result"]}],
    namespace="demo"
)

# Step 4: query back: 0 credits (cached)
rows = client.logic.tabulate(
    query="context(Key, Value)",
    namespace="demo"
)
print(f"tabulate -> {rows['cache_ref']}")

# Step 5: synthesize: 1-2 credits, data free
summary = client.prism.analyze(
    goal="What do these use cases say about who DataGrout is built for?",
    data=rows["cache_ref"],
    mode="exploratory"
)
print(f"analyze -> {summary['cache_ref']}")
print()
print(summary["result"])
```

Total: ~4–5 credits for a 5-step pipeline. None spent on moving data between steps.

## Key points

- Every tool that returns data includes a `cache_ref`
- Passing `cache_ref` as `data` is always free — no credits, no latency overhead
- Ephemerals are scoped to your account, not your process — you can pass a `cache_ref` between scripts or sessions
- Use `ephemerals.extend()` if you need a result to outlive the default TTL
- You never need to manage this explicitly — just use `result["cache_ref"]` wherever the next tool accepts `data`

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

// Step 1: search
const search = await client.perform('data-grout@1/inference.search@1', {
    query: 'DataGrout Logic Cell use cases',
    quality: 'standard'
});
console.log(`search -> ${search.cache_ref}`);

// Step 2: analyze the result, data transfer is free
const analysis = await client.perform('data-grout@1/prism.analyze@1', {
    goal: 'List the top 5 use cases with a one-sentence description each',
    data: search.cache_ref,
    mode: 'exploratory'
});
console.log(`analyze -> ${analysis.cache_ref}`);

// Step 3: further analysis, still free
const summary = await client.perform('data-grout@1/prism.analyze@1', {
    goal: 'What do these use cases say about who DataGrout is built for?',
    data: analysis.cache_ref
});
console.log(summary.result);

// Inspect an ephemeral directly
const ephemeral = await client.perform('data-grout@1/ephemerals.get@1', {
    ref: search.cache_ref
});
console.log(`${ephemeral.size_bytes} bytes, expires ${ephemeral.expires_at}`);
```

## Next steps

- [cache-ref-composition](../../patterns/cache-ref-composition/) — advanced piping patterns
- [zero-credit-pipeline](../../patterns/zero-credit-pipeline/) — building pipelines where most steps are free
- [first-flow](../first-flow/) — compose tools declaratively with flow.into
