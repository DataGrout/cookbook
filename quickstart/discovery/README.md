# Quickstart: Discovering and Composing Tools

> **Difficulty:** Beginner  
> **Credits:** 0 (discovery is free)  
> **Time:** ~5 minutes

## What it does

`discovery.discover` is the entry point for the DataGrout tool ecosystem. Instead of knowing tool names upfront, describe what you want and the discovery engine finds matching tools, ranked by semantic similarity.

```
describe a goal -> discover matching tools -> perform the best match
```

This is how agents bootstrap themselves: they don't need a hardcoded tool list. They describe their current intent and discover the right tool on the fly.

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `discovery.discover` | Find tools by natural-language goal | 0 |
| `discovery.perform` | Execute a discovered tool | 0 + tool cost |
| `discovery.plan` | Plan a multi-step workflow | 0 |
| `discovery.summary` | Compact overview of all available tools | 0 |

## The Recipe

### Step 1: Discover tools for a goal

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# Short phrase describing what you need: keep it under 10 words
results = client.perform("data-grout@1/discovery.discover@1", {
    "goal": "get invoices"
})

for tool in results["tools"]:
    print(f"{tool['tool_name']:45s}  score={tool['score']:.2f}")
    print(f"  {tool['description'][:80]}")
```

Output:
```
quickbooks@1/list_invoices@1              score=0.94
  Retrieve a list of invoices from QuickBooks Online...
quickbooks@1/get_invoice@1               score=0.87
  Retrieve a single invoice by ID...
xero@1/list_invoices@1                   score=0.81
  ...
```

### Step 2: Execute a discovered tool

```python
# Use the tool_name from discover results directly
invoice_list = client.perform("data-grout@1/discovery.perform@1", {
    "tool_name": "quickbooks@1/list_invoices@1",
    "args": {
        "status": "unpaid",
        "limit": 50
    }
})

print(invoice_list["data"])
```

`discovery.perform` is a pass-through, it routes to the actual tool and returns its result. You can also call the tool directly if you know its name.

### Step 3: Discover with integration filter

Restrict to a specific integration when you know the source:

```python
# Only look in Salesforce
leads = client.perform("data-grout@1/discovery.discover@1", {
    "goal": "find leads by company",
    "integrations": ["salesforce"]
})
```

### Step 4: Plan a multi-step workflow

`discovery.plan` maps a natural-language goal to an ordered sequence of tools:

```python
plan = client.perform("data-grout@1/discovery.plan@1", {
    "goal": "get all unpaid invoices and flag the ones overdue by more than 30 days",
    "have": {
        "customer_id": "string"
    }
})

print(plan["steps"])
# [
#   {"tool": "quickbooks@1/list_invoices@1", "args": {"status": "unpaid"}},
#   {"tool": "data-grout@1/prism.refract@1", "args": {"goal": "extract overdue > 30 days"}},
# ]

# Execute the plan via its handle
result = client.perform("data-grout@1/discovery.perform@1", {
    "skill_handle": plan["call_handle"],
    "inputs": {"customer_id": "CUST-123"}
})
```

### Step 5: Get a full tool map

`discovery.summary` returns a compact overview of all available tools, useful for agents that want to orient themselves at the start of a session:

```python
summary = client.perform("data-grout@1/discovery.summary@1", {})
print(summary["text"])
# Compact text block listing all suites and tools
# e.g.:
# logic: assert, query, reflect, constrain, tabulate, hydrate, export, import
# batteries: search, describe, install_many, installed, remove, validate
# math: describe, trend, window, outliers, normalize, correlate, rank, ...
# ...
```

Use `focus` to drill into a specific suite:

```python
logic_summary = client.perform("data-grout@1/discovery.summary@1", {
    "focus": "logic"
})
```

## Agent bootstrap pattern

An autonomous agent that starts a session without hardcoded tool names:

```python
from agentsmith import Agent  # requires Agentsmith SDK — coming soon
from datagrout_conduit import Client

client = Client(identity_auto=True)

class AdaptiveAgent(Agent):
    def on_task(self, task: str):
        # 1. Discover relevant tools
        tools = client.perform("data-grout@1/discovery.discover@1", {
            "goal": task,
            "limit": 3
        })

        if not tools["tools"]:
            return {"error": "No tools found for task"}

        best = tools["tools"][0]

        # 2. If high confidence, execute directly
        if best["score"] > 0.85:
            return client.perform("data-grout@1/discovery.perform@1", {
                "tool_name": best["tool_name"],
                "args": self.extract_args(task, best["input_contract"])
            })

        # 3. If multi-step, plan first
        plan = client.perform("data-grout@1/discovery.plan@1", {"goal": task})
        return client.perform("data-grout@1/discovery.perform@1", {
            "skill_handle": plan["call_handle"],
            "inputs": {}
        })
```

## Discovery vs hardcoding

| Approach | When to use |
|----------|-------------|
| Hardcode `tool@1/name@1` | You know the exact tool and it won't change |
| `discovery.discover` | Agent adapts to available integrations at runtime |
| `discovery.plan` | Multi-step goals where tool order matters |
| `discovery.summary` | Agent orients itself at session start |

Discovery is especially useful when:
- The agent runs against different tenants who have different integrations enabled
- You're building a general-purpose assistant that needs to pick tools at runtime
- You want to future-proof against tool additions without redeploying agent code

## Discovery with analysis

Pipe discovered results through `prism.analyze` for a semantic summary of what's available:

```python
# Find all tools matching "customer data" and get an AI summary
tools = client.perform("data-grout@1/discovery.discover@1", {
    "goal": "customer data",
    "limit": 10,
    "analyze": "What patterns exist in these tools and what's missing?"
})
print(tools["analysis"])
```

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

// Discover tools for a goal
const results = await client.perform('data-grout@1/discovery.discover@1', {
    goal: 'get invoices'
});

results.tools.forEach((tool: any) => {
    console.log(`${tool.tool_name}  score=${tool.score.toFixed(2)}`);
});

// Execute a discovered tool
const invoices = await client.perform('data-grout@1/discovery.perform@1', {
    tool_name: 'quickbooks@1/list_invoices@1',
    args: { status: 'unpaid', limit: 50 }
});
console.log(invoices.data);

// Plan a multi-step workflow
const plan = await client.perform('data-grout@1/discovery.plan@1', {
    goal: 'get all unpaid invoices and flag the ones overdue by more than 30 days',
    have: { customer_id: 'string' }
});

const planResult = await client.perform('data-grout@1/discovery.perform@1', {
    skill_handle: plan.call_handle,
    inputs: { customer_id: 'CUST-123' }
});

// Get a full tool map
const summary = await client.perform('data-grout@1/discovery.summary@1', {});
console.log(summary.text);

// Drill into a specific suite
const logicSummary = await client.perform('data-grout@1/discovery.summary@1', {
    focus: 'logic'
});
```

## See also

- [hello-world](../hello-world/) — first tool call (hardcoded name)
- [onramp-bootstrap](../../patterns/onramp-bootstrap/) — how agents register and authenticate
- [zero-credit-pipeline](../../patterns/zero-credit-pipeline/) — combine discovered tools at zero cost
