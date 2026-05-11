# Recipe: Log Processing Pipeline

> **Difficulty:** Intermediate  
> **Credits:** 2–6  
> **Time:** ~5 minutes

## What it does

Process a batch of server logs through `flow.into` with a conditional routing step: classify entries by severity, then dispatch `critical` logs to forensic investigation and `warning` logs to a digest summary. The pipeline handles a mixed log stream in one call and returns structured output per severity tier.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `prism.refract` | 1 | Classify log entries by severity |
| `flow.route` | 0 | Dispatch to different tools based on severity |
| `forensic.investigate` | 2–4 | Root-cause analysis on critical errors |
| `prism.analyze` | 1–2 | Warning digest and pattern summary |

## The Recipe

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

logs = """
2026-05-10 03:12:44 ERROR  [auth]     JWT validation failed, token expired (user_id=u_001)
2026-05-10 03:12:51 WARN   [rate]     Request rate 92% of limit for tenant=example-co
2026-05-10 03:13:02 ERROR  [db]       Connection pool exhausted, 0 connections available
2026-05-10 03:13:05 ERROR  [db]       Query timeout after 30s (query=user_lookup, tenant=example-co)
2026-05-10 03:13:08 WARN   [cache]    Cache miss rate 78% (threshold: 60%)
2026-05-10 03:13:15 ERROR  [payment]  Stripe webhook 500, downstream unavailable
"""

result = client.perform("data-grout@1/flow.into@1", {
    "plan": [
        # Step 1: classify and structure the raw log text
        {
            "tool": "data-grout@1/prism.refract@1",
            "output": "classified",
            "args": {
                "goal": "Extract each log entry as {timestamp, level, service, message}. "
                        "Normalize level to: critical (ERROR affecting user-facing systems), "
                        "warning (WARN or degraded-but-not-broken), or info.",
                "data": logs
            }
        },
        # Step 2: route based on worst severity in the batch
        {
            "type": "conditional",
            "input": "classified",
            "branches": [
                {
                    "when": [{"field": "entries", "op": "contains", "value": "critical"}],
                    "label": "has_criticals",
                    "then": {
                        "tool": "data-grout@1/forensic.investigate@1",
                        "args": {
                            "data": "$classified",
                            "goal": "Identify the root cause of the critical errors and the blast radius. "
                                    "Which service failed first? What cascaded from it?"
                        }
                    }
                }
            ],
            "else": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {
                    "goal": "Summarize the warnings. Are any trending toward a critical threshold?",
                    "data": "$classified",
                    "mode": "causal"
                }
            },
            "output": "routed"
        }
    ]
})

print(result["execution_result"]["routed"])
```

## Standalone flow.route

Use `flow.route` directly when you already have classified data and just need to dispatch it:

```python
# Assumes `classified_ref` is a cache_ref from a prior prism.refract call
dispatch = client.perform("data-grout@1/flow.route@1", {
    "cache_ref": classified_ref,
    "branches": [
        {
            "when": [{"field": "level", "op": "eq", "value": "critical"}],
            "label": "critical",
            "then": {
                "tool": "data-grout@1/forensic.investigate@1",
                "args": {"goal": "Root cause and blast radius"}
            }
        },
        {
            "when": [{"field": "level", "op": "eq", "value": "warning"}],
            "label": "warning",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {"goal": "Warning pattern summary", "mode": "exploratory"}
            }
        }
    ],
    "else": "prism.analyze"   # tool name shorthand — payload forwarded as-is
})

print(dispatch["matched"])    # True
print(dispatch["label"])      # "critical"
print(dispatch["result"])     # forensic investigation output
```

## Multi-branch: three severity tiers

```python
client.perform("data-grout@1/flow.route@1", {
    "cache_ref": classified_ref,
    "branches": [
        {
            "when": [{"field": "level", "op": "eq", "value": "critical"}],
            "label": "page_oncall",
            "then": {
                "tool": "data-grout@1/forensic.investigate@1",
                "args": {"goal": "Root cause, ETA to resolution, who to page"}
            }
        },
        {
            "when": [{"field": "level", "op": "eq", "value": "warning"}],
            "label": "digest",
            "then": {
                "tool": "data-grout@1/prism.analyze@1",
                "args": {"goal": "Warning digest for next standup", "mode": "exploratory"}
            }
        },
        {
            "when": "_",   # catch-all
            "label": "info_skip",
            "then": "prism.refract",   # just pass through structured
        }
    ]
})
```

## Asserting results into a logic cell

After processing, store the findings for later querying:

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

# After the pipeline above, assert the incident into your LC
client.perform("data-grout@1/logic.assert@1", {
    "facts": [
        {"type": "entity", "name": "incident_20260510", "attributes": {
            "root_cause": "db_pool_exhaustion",
            "first_failure": "db",
            "cascaded_to": "payment",
            "severity": "critical"
        }},
        {"type": "relation", "subject": "incident_20260510", "relation": "triggered_by",
         "object": "acme_rate_spike"},
    ],
    "namespace": "incidents"
})

# Query later
client.perform("data-grout@1/logic.query@1", {
    "query": "incident_severity(X, critical)",
    "namespace": "incidents"
})
```

## Variations

**Real-time log tail (WebSocket):**
```python
from datagrout_conduit import Client

async with Client("wss://app.datagrout.ai/servers/<uuid>/rpc", transport="websocket") as client:
    async for event in client.subscribe("log.stream", {"filter": "level:error"}):
        # Route each incoming log event through the pipeline
        await client.perform("data-grout@1/flow.route@1", {
            "payload": event,
            "branches": [...]
        })
```

**Scheduled log digest:**
```python
# Save as a skill and schedule it
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [...],
    "save_as_skill": True,
    "skill_name": "hourly-log-digest",
    "schedule": "every hour"
})
```

## See also

- [conditional-routing](../conditional-routing/) — `flow.route` patterns in depth
- [trend-analysis-pipeline](../trend-analysis-pipeline/) — flow.into for time-series data
- [causal-chain-tracing](../../forensic/causal-chain-tracing/) — forensic investigation in detail
