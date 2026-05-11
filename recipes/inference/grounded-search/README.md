# Recipe: Grounded Search

## What it does

`inference.search` retrieves current, cited information from the web and returns it with source URLs. Unlike an LLM knowledge base, results are live and traceable. `inference.research` runs multiple questions in parallel for deeper coverage. Both return a `cache_ref` for zero-cost downstream piping.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `inference.search` | 1–3 | Single grounded query |
| `inference.research` | 3–10 | Multi-question parallel research |
| `inference.rfi` | 5–15 | Deep structured research (request for information) |

## Credits cost

`inference.search`: 1 credit for standard quality, up to 3 for high. `inference.research` scales with question count and concurrency.

## The Recipe

### Basic search

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

result = client.inference.search(
    query="DataGrout Logic Cell pricing and feature comparison 2026",
    quality="standard"  # "standard" | "high"
)

print(result["result"])       # synthesized answer
print(result["citations"])    # list of {"url": ..., "title": ..., "snippet": ...}
print(result["cache_ref"])    # pass to downstream tools at zero cost
```

### Quality levels

```python
# Standard (1 credit): fast, good for factual lookups
result = client.inference.search(query="...", quality="standard")

# High (2–3 credits): deeper synthesis, more citations, better for analysis
result = client.inference.search(query="...", quality="high")
```

### Multi-question research

`inference.research` runs questions in parallel and returns a structured answer per question. Auto-promotes to a background task when question count > 2 or quality is high.

```python
result = client.inference.research(
    questions=[
        "What are the main competitors to DataGrout in the AI reasoning space?",
        "What is the pricing model for LangChain and LlamaIndex?",
        "Which enterprise customers are publicly using symbolic reasoning tools?"
    ],
    quality="high",
    concurrency=3  # run all in parallel
)

# Each question has its own answer and citations
for q in result["answers"]:
    print(f"Q: {q['question']}")
    print(f"A: {q['answer'][:200]}...")
    print(f"Sources: {len(q['citations'])} citations")
    print()
```

### Deep structured research (RFI)

For comprehensive coverage of a topic, analogous to a research brief. Returns sections, not just answers.

```python
result = client.inference.rfi(
    topic="DataGrout competitive landscape in enterprise AI tooling",
    sections=[
        "Market overview and key players",
        "Pricing and packaging comparison",
        "Technical differentiation: symbolic vs vector approaches",
        "Customer segment analysis",
        "Recent funding and M&A activity"
    ],
    quality="high"
)

for section in result["sections"]:
    print(f"## {section['title']}")
    print(section["content"])
    print()
```

## Piping to downstream tools

Search results are large. Use `cache_ref` instead of passing raw text.

```python
# Search
search = client.inference.search(
    query="Q1 2026 SaaS funding rounds over $50M",
    quality="high"
)

# Normalize: passes cache_ref, not raw text
facts = client.prism.refract(
    goal="Extract: company, round_type, amount_usd, lead_investor, date",
    data=search["cache_ref"]  # zero-cost pass-through
)

# Store
client.logic.assert_facts(facts=facts["result"], namespace="funding_tracker")

# Analyze
analysis = client.prism.analyze(
    goal="Which sectors are attracting the most Series B capital?",
    data=facts["cache_ref"]
)
```

## Including citations in LC facts

```python
search = client.inference.search(query="...")

# Store the top citation alongside the findings
client.logic.assert_facts(facts=[
    {"type": "context", "key": "source_url", "value": search["citations"][0]["url"]},
    {"type": "context", "key": "source_title", "value": search["citations"][0]["title"]},
], namespace="my_research")
```

## Filtering and re-querying

```python
# Search is not a filter: if results seem off, narrow the query
# Bad: broad query, lots of noise
result = client.inference.search(query="AI tools")

# Good: specific query, targeted results
result = client.inference.search(
    query="Prolog-based reasoning tools for enterprise AI agents 2025 2026"
)

# For time-sensitive queries, include the year
result = client.inference.search(
    query="DataGrout product updates May 2026"
)
```

## Background task handling

`inference.research` with multiple high-quality questions auto-promotes:

```python
result = client.inference.research(questions=[...], quality="high")

if "task_id" in result:
    # Promoted to background — wait for it
    final = client.tasks.wait(task_id=result["task_id"], timeout=300)
    answers = final["output"]["answers"]
else:
    answers = result["answers"]
```

## See also

- [background-tasks](../../../patterns/background-tasks/) — handling auto-promoted research tasks
- [research-to-report](../../combined/research-to-report/) — search -> refract -> assert -> analyze pipeline
- [competitive-intel-pipeline](../../combined/competitive-intel-pipeline/) — parallel research at scale
