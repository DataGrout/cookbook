# Recipe: Conditional Routing with flow.route

> **Difficulty:** Intermediate  
> **Credits:** varies by branch  
> **Time:** ~5 minutes

## What it does

`flow.route` dispatches a payload to different tools based on ordered predicate conditions, first match wins. Use it standalone to branch on pre-classified data, or embed it as a `conditional` step inside `flow.into` to branch mid-pipeline.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `flow.route` | 0 | Evaluate predicates and dispatch to matching tool |
| Any target tool | varies | Whatever branch runs |

## The `when` forms

`flow.route` accepts four condition styles:

```python
# 1. Single predicate
{"when": {"field": "score", "op": "gte", "value": 90}}

# 2. AND of multiple predicates (all must match)
{"when": [
    {"field": "tier", "op": "eq", "value": "enterprise"},
    {"field": "arr", "op": "gt", "value": 100000}
]}

# 3. Truthy path shorthand: non-null, non-empty, non-zero
{"when": "$errors"}           # matches if the 'errors' field is truthy

# 4. Catch-all (like else)
{"when": "_"}                 # or True, always matches
```

Supported operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `not_in`, `contains`, `starts_with`, `ends_with`, `is_null`, `not_null`.

## Basic example: customer tier routing

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

customer = {
    "id": "cust_8821",
    "tier": "enterprise",
    "arr": 450000,
    "open_tickets": 3,
    "last_contact_days": 42
}

result = client.perform("data-grout@1/flow.route@1", {
    "payload": customer,
    "branches": [
        {
            "when": [
                {"field": "tier", "op": "eq", "value": "enterprise"},
                {"field": "last_contact_days", "op": "gt", "value": 30}
            ],
            "label": "at_risk_enterprise",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Draft a re-engagement action plan for this enterprise customer",
                    "data": "$payload"
                }
            }
        },
        {
            "when": [{"field": "open_tickets", "op": "gt", "value": 5}],
            "label": "high_ticket_volume",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Summarize the support situation and recommend escalation path",
                    "data": "$payload"
                }
            }
        },
        {
            "when": "_",
            "label": "standard",
            "then": "prism.refract"   # just structure the data, no LLM analysis
        }
    ]
})

print(result["matched"])   # True
print(result["label"])     # "at_risk_enterprise"
print(result["result"])    # prism.analyze output
```

## Inside flow.into: mid-pipeline branching

Embed routing as a `conditional` step type inside a flow plan:

```python
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [
        # Step 1: extract structured data from raw input
        {
            "tool": "data-grout@1/prism.refract@1",
            "output": "structured",
            "args": {
                "goal": "Extract: customer_id, sentiment (positive/neutral/negative), "
                        "issue_type, urgency (1-5)",
                "data": support_ticket_text
            }
        },
        # Step 2: branch on urgency + sentiment
        {
            "type": "conditional",
            "input": "structured",
            "output": "response",
            "branches": [
                {
                    "when": [
                        {"field": "urgency", "op": "gte", "value": 4},
                        {"field": "sentiment", "op": "eq", "value": "negative"}
                    ],
                    "label": "escalate",
                    "then": {
                        "tool": "data-grout@1/prism.analyze@1",
                        "args": {
                            "goal": "Write an empathetic response acknowledging urgency, "
                                    "commit to resolution timeline, and flag for senior support",
                            "data": "$structured",
                            "mode": "deductive"
                        }
                    }
                },
                {
                    "when": [{"field": "urgency", "op": "lte", "value": 2}],
                    "label": "self_service",
                    "then": {
                        "tool": "data-grout@1/inference.search@1",
                        "args": {
                            "query": "$structured.issue_type",
                            "goal": "Find relevant documentation links"
                        }
                    }
                }
            ],
            "else": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Write a standard helpful response",
                    "data": "$structured"
                }
            }
        }
    ]
})
```

## Routing on cache_ref (large data)

When the payload is large, pass a `cache_ref` from a prior tool call instead of re-transmitting the data:

```python
# Previous tool returned a cache_ref
prior_result = client.perform("data-grout@1/prism.refract@1", {
    "goal": "Classify documents by type",
    "data": large_document_batch
})
ref = prior_result["_dg"]["cache_ref"]

# Route without re-sending the data
client.perform("data-grout@1/flow.route@1", {
    "cache_ref": ref,
    "branches": [
        {"when": [{"field": "type", "op": "eq", "value": "contract"}], "then": "..."},
        {"when": [{"field": "type", "op": "eq", "value": "invoice"}],  "then": "..."},
        {"when": "_", "then": "prism.analyze"}
    ]
})
```

## `else` and unmatched branches

If no branch matches and there's no `else`, `flow.route` returns `{matched: false, branch: -1}` rather than raising an error. Always add a catch-all `"_"` branch or an `else` tool if you need guaranteed output.

```python
# Safe pattern: explicit catch-all
{
    "when": "_",
    "label": "unhandled",
    "then": {
        "tool": "data-grout@1/prism.analyze@1",
        "args": {"goal": "Describe what this data is and what to do with it", "data": "$payload"}
    }
}
```

## See also

- [log-processing](../log-processing/) — flow.route in a full log classification pipeline
- [trend-analysis-pipeline](../trend-analysis-pipeline/) — using conditional steps for alert branching
- [background-tasks](../../../patterns/background-tasks/) — run branches asynchronously
