# Recipe: Persistent Documents

> **Difficulty:** Beginner  
> **Credits:** 1 per create or update  
> **Time:** ~10 minutes

## What it does

The docs suite gives agents a persistent notepad that survives across sessions. Unlike ephemeral `cache_ref` results (which expire) or LC facts (which are structured/queryable), docs are free-form text, markdown, checklists, JSON, that you create, update, retrieve, and delete by `ref`.

Common uses:
- **Agent working memory** — create a doc at the start of a long task, update it as you go, delete when done
- **Research notes** — persist findings during a multi-session investigation
- **Shared reports** — create a doc an agent produces; a human reads it later
- **Cleanup lists** — track what an agent has done or plans to do next

## Tools used

| Tool | Purpose | Credits |
|------|---------|---------|
| `docs.create` | Create a new persistent document | 1 |
| `docs.update` | Overwrite or append to an existing document | 1 |
| `docs.get` | Retrieve a document by ref | 0 |
| `docs.list` | List documents, optionally filtered by tag or format | 0 |
| `docs.delete` | Permanently delete a document | 0 |

## The Recipe

### Create a document

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

doc = client.perform("data-grout@1/docs.create@1", {
    "title": "Q2 Research Notes",
    "body": "## Sources\n\n- TBD\n\n## Key Findings\n\n",
    "format": "markdown",
    "tags": ["project:q2-research", "status:in-progress"]
})

ref = doc["ref"]    # "doc_a1b2c3...", save this for later calls
print(ref)
```

Use `tags` to group related documents. Namespaced tags like `project:q2-research` or `session:2026-05-11` make bulk operations easy later.

### Retrieve and update

```python
# Get the full document
doc = client.perform("data-grout@1/docs.get@1", {"ref": ref})
print(doc["body"])
print(f"Version: {doc['version']}")   # increments on every update

# Update: overwrite the body
client.perform("data-grout@1/docs.update@1", {
    "ref": ref,
    "body": doc["body"] + "\n## New Section\n\nAdded during session 2.",
    "tags": ["project:q2-research", "status:complete"]
})
```

Updates are full overwrites of the fields you pass. Fields you omit are unchanged.

### List documents

```python
# All docs for a project
docs = client.perform("data-grout@1/docs.list@1", {
    "tags": ["project:q2-research"]
})

for d in docs["docs"]:
    print(f"{d['ref']}  {d['title']}  v{d['version']}  {d['updated_at']}")
```

Filter by format, tag, or free-text search:

```python
# Find all markdown docs with "invoice" in the title
docs = client.perform("data-grout@1/docs.list@1", {
    "format": "markdown",
    "search": "invoice"
})
```

### Delete a document

```python
result = client.perform("data-grout@1/docs.delete@1", {"ref": ref})
print(result["status"])   # "deleted" or "not_found"
```

Deletion is permanent. Use `docs.list` or `docs.get` to confirm the ref before calling.

---

## Pattern: Agent working memory

Agents often produce intermediate results that need to survive between LLM calls but don't need to be structured facts. A working doc acts as a scratchpad:

```python
# Start of a long research task
scratchpad = client.perform("data-grout@1/docs.create@1", {
    "title": f"Research scratchpad — {task_id}",
    "body": f"# Task\n\n{task_description}\n\n# Progress\n\n",
    "format": "markdown",
    "tags": [f"session:{session_id}", "type:scratchpad"]
})
scratchpad_ref = scratchpad["ref"]

# ... agent does work, periodically appends findings ...
for finding in findings:
    doc = client.perform("data-grout@1/docs.get@1", {"ref": scratchpad_ref})
    client.perform("data-grout@1/docs.update@1", {
        "ref": scratchpad_ref,
        "body": doc["body"] + f"\n- {finding}"
    })

# End of task: retrieve final state, delete scratchpad
final = client.perform("data-grout@1/docs.get@1", {"ref": scratchpad_ref})
client.perform("data-grout@1/docs.delete@1", {"ref": scratchpad_ref})

# Optionally promote to a permanent doc
client.perform("data-grout@1/docs.create@1", {
    "title": f"Research findings — {task_id}",
    "body": final["body"],
    "tags": ["type:findings", f"task:{task_id}"]
})
```

---

## Pattern: Cleanup with approval gate

Use `flow.request-approval` before bulk-deleting documents. The tool pauses execution and waits for a human to approve via the Mission Control UI before the agent proceeds.

```python
# 1. Find all session scratchpads older than today
old_docs = client.perform("data-grout@1/docs.list@1", {
    "tags": ["type:scratchpad", "session:2026-05-10"]
})

refs_to_delete = [d["ref"] for d in old_docs["docs"]]

if not refs_to_delete:
    print("Nothing to clean up.")
else:
    # 2. Request approval before deleting
    approval = client.perform("data-grout@1/flow.request-approval@1", {
        "action": "bulk_delete_docs",
        "details": {
            "count": len(refs_to_delete),
            "refs": refs_to_delete,
            "titles": [d["title"] for d in old_docs["docs"]]
        },
        "reason": f"Cleaning up {len(refs_to_delete)} session scratchpad(s) from 2026-05-10"
    })

    # approval["status"] == "pending" — user approves via Mission Control UI
    # Your workflow layer polls /api/approvals/{approval_id} or receives a webhook
    approval_id = approval["approval_id"]
    print(f"Approval requested: {approval_id}")
    print("Waiting for approval in Mission Control...")

    # 3. When approved (your polling / webhook handler):
    def on_approved():
        for ref in refs_to_delete:
            result = client.perform("data-grout@1/docs.delete@1", {"ref": ref})
            print(f"Deleted {ref}: {result['status']}")
```

`flow.request-approval` is designed for exactly this: before any destructive or irreversible action in an agent pipeline, surface it to a human with full context. The agent holds the refs; the human sees the titles and count; deletion only runs after explicit approval.

---

## Pattern: Checklist doc for multi-step tasks

The `checklist` format renders as an interactive checklist in Mission Control:

```python
# Create a task checklist
checklist = client.perform("data-grout@1/docs.create@1", {
    "title": "Deploy checklist — v2.4.0",
    "body": "- [ ] Run migration\n- [ ] Smoke test staging\n- [ ] Update changelog\n- [ ] Tag release\n- [ ] Notify team",
    "format": "checklist",
    "tags": ["type:checklist", "release:v2.4.0"]
})

# As steps complete, update the body to tick boxes
client.perform("data-grout@1/docs.update@1", {
    "ref": checklist["ref"],
    "body": "- [x] Run migration\n- [x] Smoke test staging\n- [ ] Update changelog\n- [ ] Tag release\n- [ ] Notify team"
})
```

---

## TypeScript

```typescript
import { Client } from '@datagrout/conduit';

const client = new Client({ apiKey: 'your-key' });

// Create
const doc = await client.perform('data-grout@1/docs.create@1', {
    title: 'Q2 Research Notes',
    body: '## Sources\n\n- TBD\n\n',
    format: 'markdown',
    tags: ['project:q2-research']
});
const ref = doc.ref;

// Update
const existing = await client.perform('data-grout@1/docs.get@1', { ref });
await client.perform('data-grout@1/docs.update@1', {
    ref,
    body: existing.body + '\n## New findings\n\n...'
});

// List
const list = await client.perform('data-grout@1/docs.list@1', {
    tags: ['project:q2-research']
});
list.docs.forEach((d: any) => console.log(d.ref, d.title));

// Delete
await client.perform('data-grout@1/docs.delete@1', { ref });
```

---

## See also

- [cache-ref-composition](../../patterns/cache-ref-composition/) — ephemerals vs persistent docs
- [research-to-report](../combined/research-to-report/) — research pipeline that writes findings to a doc
- [flow/conditional-routing](../flow/conditional-routing/) — route based on doc content
