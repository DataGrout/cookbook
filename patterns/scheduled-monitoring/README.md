# Pattern: Scheduled Monitoring

## What it does

Run a monitoring check on a recurring schedule using Governor. Each firing asserts findings into an LC namespace so results accumulate across sessions and can be queried later.

## Tools used

- `governor.schedule` — create a recurring trigger
- `inference.search` or `logic.query` — the check itself
- `logic.assert` — persist findings per run
- `tasks.list` — inspect scheduled jobs

## Credits cost

Per firing: depends on tools called. Scheduling itself: 0.

## The Pattern

Governor runs a prompt on your schedule. The prompt executes like any Claude Code session, it can call DG tools, write to LC namespaces, and produce output.

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Create a recurring monitoring job
job = client.governor.schedule(
    prompt="""
    Search for news about DataGrout competitors from the past 24 hours.
    For each new development found, assert a fact into the 'competitor_alerts' namespace:
      - entity: competitor name
      - attribute: alert_type, value: the type of news (funding/product/partnership)
      - attribute: summary, value: one-sentence description
      - attribute: detected_at, value: today's date
    Then query all high-threat competitors and flag any that had activity today.
    """,
    schedule="0 8 * * *",  # 8am daily
    name="competitor-daily-monitor"
)

print(f"Job created: {job['id']}")
print(f"Next run: {job['next_run_at']}")
```

### Querying accumulated findings

```python
# Next session: all prior runs are queryable
alerts = client.logic.query(
    query="attribute(Competitor, alert_type, Type)",
    namespace="competitor_alerts"
)

# Find all funding events
funding = client.logic.query(
    query="attribute(Competitor, alert_type, funding)",
    namespace="competitor_alerts"
)

# Count alerts per competitor
table = client.logic.tabulate(
    query="attribute(Competitor, alert_type, Type)",
    namespace="competitor_alerts",
    columns=["Competitor", "Type"]
)
```

### Managing scheduled jobs

```python
# List all active jobs
jobs = client.governor.list_schedules()
for j in jobs["jobs"]:
    print(f"{j['name']}: {j['schedule']} — next: {j['next_run_at']}")

# Pause a job
client.governor.pause_schedule(job_id=job["id"])

# Resume
client.governor.resume_schedule(job_id=job["id"])

# Delete
client.governor.delete_schedule(job_id=job["id"])
```

## Common scheduling patterns

| Goal | Cron | Example |
|------|------|---------|
| Daily at 8am | `0 8 * * *` | Morning news digest |
| Hourly | `0 * * * *` | Uptime / SLA check |
| Every 15 minutes | `*/15 * * * *` | Error rate monitoring |
| Weekly Monday 9am | `0 9 * * 1` | Weekly report |
| First of month | `0 9 1 * *` | Monthly rollup |

## Monitoring with threshold alerts

```python
# Schedule a check that alerts when a metric crosses a threshold
client.governor.schedule(
    prompt="""
    Query the 'api_metrics' namespace for error rates in the last hour:
      logic.query("attribute(endpoint, error_rate_pct, Rate), Rate > 5.0")
    
    If any endpoints exceed 5% error rate, assert an alert fact:
      entity: the endpoint
      attribute: alert_fired, value: error_rate_exceeded
      attribute: rate, value: the actual rate
    
    Then call inference.search to investigate any known incidents.
    """,
    schedule="*/15 * * * *",
    name="error-rate-monitor"
)
```

## Idempotent fact patterns

Use timestamped entity names to avoid overwriting prior runs:

```python
# In your scheduled prompt, use dated entity names
from datetime import date

today = date.today().isoformat()
entity_name = f"alert_{today}_{competitor_name}"

client.logic.assert_facts(facts=[
    {"type": "entity", "name": entity_name},
    {"type": "attribute", "entity": entity_name, "attribute": "competitor", "value": competitor_name},
    {"type": "attribute", "entity": entity_name, "attribute": "date", "value": today},
    {"type": "attribute", "entity": entity_name, "attribute": "summary", "value": summary},
], namespace="competitor_alerts")
```

## See also

- [background-tasks](../background-tasks/) — non-blocking long-running calls
- [competitive-intel-pipeline](../../recipes/combined/competitive-intel-pipeline/) — pipeline this pattern builds on
- [research-agent](../../agents/research-agent/) — FSM agent version of the same idea
