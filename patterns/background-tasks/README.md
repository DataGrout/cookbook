# Pattern: Background Tasks

## What it does

Trigger a long-running `inference.research` call, observe automatic background task promotion, use `tasks.wait` and `tasks.result` to retrieve output. The non-blocking pattern for expensive operations.

## Tools used

- `inference.research` — multi-question deep research (auto-promotes to background)
- `tasks.list` — list active background tasks
- `tasks.wait` — block until a task completes
- `tasks.result` — retrieve task output

## Credits cost

Varies. Background promotion is automatic for tasks estimated >30 seconds.

## The Recipe

Some tools automatically run as background tasks when the estimated runtime exceeds a threshold. You don't need to opt in, DG detects it and returns a `task_id` instead of a result.

```python
from datagrout_conduit import Client
import time

client = Client(api_key="your-key")

# This call will auto-promote to background (multiple questions, high quality)
response = client.inference.research(
    questions=[
        "What are the main use cases for DataGrout in enterprise environments?",
        "How does DataGrout compare to vector databases for knowledge storage?",
        "What is the typical ROI for teams using symbolic reasoning over LLM-only approaches?"
    ],
    quality="high",
    concurrency=3
)

# Check if it ran inline or was promoted
if "task_id" in response:
    task_id = response["task_id"]
    print(f"Promoted to background task: {task_id}")
    print(f"Estimated time: {response['estimated_seconds']}s")

    # Non-blocking: do other work while research runs
    while True:
        tasks = client.tasks.list()
        task = next((t for t in tasks["tasks"] if t["id"] == task_id), None)
        print(f"Status: {task['status']} ({task['progress_pct']}%)")
        if task["status"] in ("completed", "failed"):
            break
        time.sleep(5)

    # Retrieve result
    result = client.tasks.result(task_id=task_id)
    print(result["output"])
else:
    # Ran inline (short task)
    print(response["result"])
```

### Simplified with tasks.wait

```python
response = client.inference.research(questions=[...], quality="high")

if "task_id" in response:
    # Block until done (polling handled internally)
    result = client.tasks.wait(task_id=response["task_id"], timeout=300)
    print(result["output"])
```

## Task lifecycle

```
Submitted -> Queued -> Running -> Completed
                             ↘ Failed
                             ↘ Cancelled
```

```python
# Cancel a running task
client.tasks.cancel(task_id=task_id)

# List all tasks (active and recent)
tasks = client.tasks.list(status="running")
```

## WebSocket subscription for real-time updates

```python
# Subscribe to task events (requires WebSocket transport)
ws_client = Client(api_key="your-key", transport="websocket")
sub = ws_client.subscribe(topic=f"tasks.{task_id}.events")

async for event in sub:
    print(f"Progress: {event['progress_pct']}% — {event['status_message']}")
    if event["status"] in ("completed", "failed"):
        break
```

## Which tools auto-promote?

| Tool | Promotes when |
|------|--------------|
| `inference.research` | questions > 2 or quality = "high" |
| `inference.rfi` | always |
| `invariant.lens` | codebase > 20 files |
| `prism.analyze` | mode = "deep" and data > 10k chars |

## Variations

**Parallel background tasks:**
```python
# Fire multiple research tasks simultaneously
task_ids = []
for topic in topics:
    r = client.inference.research(questions=[topic], quality="high")
    if "task_id" in r:
        task_ids.append(r["task_id"])

# Wait for all
results = [client.tasks.wait(task_id=tid) for tid in task_ids]
```

## See also

- [scheduled-monitoring](../scheduled-monitoring/) — background tasks on a schedule
- [competitive-intel-pipeline](../../recipes/combined/competitive-intel-pipeline/) — parallel research tasks
