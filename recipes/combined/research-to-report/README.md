# Recipe: Research to Report

> **Difficulty:** Intermediate  
> **Credits:** 8–20  
> **Time:** ~15 minutes

## What it does

Gather grounded research on a topic, structure the findings as LC facts, then synthesize a polished report. The zero-credit compute steps (tabulate, group) compress the raw research before handing it to a single analysis call, keeping cost proportional to the final output, not the raw data volume.

Pipeline: `inference.search` -> `prism.refract` -> `logic.assert` -> `logic.tabulate` -> `prism.analyze`

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `inference.search` × N | 1–3 each | Grounded web research per sub-question |
| `prism.refract` | 1 | Normalize raw research to structured facts |
| `logic.assert` | 1 | Persist findings across sessions |
| `logic.tabulate` | 0 | Flatten facts to queryable table |
| `prism.analyze` | 2–5 | Final report synthesis |

## The Recipe

```python
from datagrout_conduit import Client

client = Client("https://app.datagrout.ai/servers/<uuid>/mcp")

topic = "adoption of AI coding assistants in enterprise software teams"
namespace = "research_ai_coding"

# Step 1: Multi-angle research
questions = [
    f"What is the current adoption rate of AI coding assistants in enterprise teams? {topic}",
    f"What are the main blockers to enterprise adoption of AI coding tools? {topic}",
    f"Which vendors lead the market and what are their differentiators? {topic}",
    f"What ROI metrics do enterprises report after adopting AI coding assistants? {topic}",
]

raw_results = []
for q in questions:
    result = client.perform("data-grout@1/inference.search@1", {
        "query": q,
        "quality": "high"
    })
    raw_results.append(result["result"])

combined_research = "\n\n---\n\n".join(raw_results)

# Step 2: Structure as facts
structured = client.perform("data-grout@1/prism.refract@1", {
    "goal": """Extract findings as entity/attribute facts:
    - Each finding is an entity with a slug name (e.g., finding_adoption_rate)
    - Attributes: category (adoption/blocker/vendor/roi), claim, evidence, confidence (high/medium/low)
    - Vendors get their own entities with: market_position, key_differentiator, pricing_model
    - Output as a JSON array of LC fact objects.""",
    "data": combined_research
})

# Step 3: Persist (survives session end)
client.perform("data-grout@1/logic.assert@1", {
    "namespace": namespace,
    "facts": structured["result"]
})

# Step 4: Tabulate for analysis
table = client.perform("data-grout@1/logic.tabulate@1", {
    "namespace": namespace,
    "query": "attribute(Finding, Attr, Value)",
    "columns": ["Finding", "Attr", "Value"]
})

# Step 5: Report
report = client.perform("data-grout@1/prism.analyze@1", {
    "goal": f"""Write a structured research report on: {topic}

    Structure:
    1. Executive Summary (3 sentences)
    2. Adoption Landscape — current rates, trajectory, key drivers
    3. Blockers and Friction — ranked by frequency in research
    4. Vendor Landscape — top players, differentiation, pricing patterns
    5. ROI Evidence — quantified where available, anecdotal otherwise
    6. Recommendations — 3 actionable recommendations for an enterprise considering adoption

    Cite confidence levels where evidence is mixed.""",
    "data": table["cache_ref"],
    "mode": "causal"
})

print(report["answer"])
```

## Inside flow.into

The same pipeline as a single call:

```python
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [
        {
            "tool": "data-grout@1/inference.search@1",
            "output": "raw",
            "args": {
                "query": f"enterprise adoption of AI coding assistants — rates, blockers, ROI",
                "quality": "high"
            }
        },
        {
            "tool": "data-grout@1/prism.refract@1",
            "output": "structured",
            "args": {
                "goal": "Extract findings as {category, claim, evidence, confidence} objects",
                "data": "$raw"
            }
        },
        {
            "tool": "data-grout@1/logic.assert@1",
            "output": "asserted",
            "args": {
                "namespace": "research_ai_coding",
                "facts": "$structured"
            }
        },
        {
            "tool": "data-grout@1/prism.analyze@1",
            "output": "report",
            "args": {
                "goal": "Write a structured 5-section research report with executive summary",
                "data": "$structured",
                "mode": "causal"
            }
        }
    ],
    "refract": "Extract: title, executive_summary, top_3_findings, top_3_recommendations"
})

print(result["execution_result"]["report"])
print(result["execution_result"]["_refract"])
```

## Querying the persisted findings later

```python
# Next session: no re-research needed

# All high-confidence findings
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "research_ai_coding",
    "query": "attribute(Finding, confidence, high), attribute(Finding, category, Category)"
})

# All vendors
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "research_ai_coding",
    "query": "attribute(Vendor, market_position, Position)"
})

# Natural language
result = client.perform("data-grout@1/logic.query@1", {
    "namespace": "research_ai_coding",
    "query": "What are the main blockers to adoption?",
    "mode": "semantic"
})
```

## Variations

**Multi-source comparison**, search the same question against different source types then compare:
```python
sources = ["analyst reports", "developer forums", "vendor case studies"]
results = [
    client.perform("data-grout@1/inference.search@1", {
        "query": f"{topic} site:{s}",
        "quality": "high"
    })
    for s in sources
]
```

**Incremental refresh**, add new findings without re-running the whole pipeline:
```python
# Run inference.search on new developments only
new_research = client.perform("data-grout@1/inference.search@1", {
    "query": f"{topic} news since May 2026",
    "quality": "standard"
})
# Assert into the same namespace: existing facts are preserved
client.perform("data-grout@1/logic.assert@1", {
    "namespace": namespace,
    "facts": new_research["result"]
})
```

**Scheduled refresh:**
```python
result = client.perform("data-grout@1/flow.into@1", {
    "plan": [...],
    "save_as_skill": True,
    "skill_name": "monthly-research-refresh",
    "schedule": "first monday of each month"
})
```

## See also

- [competitive-intel-pipeline](../competitive-intel-pipeline/) — research -> threat matrix variation
- [assert-and-query](../../logic/assert-and-query/) — LC fact types and query patterns
- [trend-analysis-pipeline](../../flow/trend-analysis-pipeline/) — quantitative data version
