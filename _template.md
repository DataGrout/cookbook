# Recipe: [Name]

> **Difficulty:** Beginner / Intermediate / Advanced  
> **Time:** ~N minutes  
> **Credits:** ~N (highlight zero-credit steps)

## What it does

One-paragraph description of the recipe and what problem it solves.

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `data-grout@1/tool.name@1` | What it does | 0 / ~N |

## The Recipe

### Step 1: [Action]

```python
# Python example
result = await client.perform("tool.name", args={...})
```

```typescript
// TypeScript
const result = await client.perform({ tool: 'tool.name', args: {...} });
```

### Step 2: [Action]

```python
result2 = await client.perform("tool.name2", args={
    "input": result["cache_ref"]  # pipe from step 1
})
```

## Try it

Copy-pasteable minimal example that runs end-to-end:

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")
# Or MCP URL (default for Claude Code):
# client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# ... full working example ...
```

## Variations

- **Variation A:** Alternative approach for [scenario]
- **Variation B:** Extension for [use case]

## See also

- [Related recipe](../other/recipe.md)
- [Benchmark result](https://github.com/DataGrout/benchmarks/tree/main/benchmarks/NN-name)
