# Recipe: Analyze Modes

## What it does

`prism.analyze` synthesizes structured findings from data using a natural language goal. The `mode` parameter controls the analytical lens, exploratory, competitive, causal, or deep. Each mode shapes how the model reasons over the data, not just how it formats the output.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `prism.analyze` | 1–5 | Semantic analysis with configurable reasoning mode |

## Credits cost

Standard modes (exploratory, competitive, causal): 1–3 credits. `mode="deep"` with large data: 3–5 credits. Cached results: 0.

## Modes

| Mode | Best for | Reasoning style |
|------|----------|----------------|
| `exploratory` | Open-ended discovery, unknown data | Surfaces patterns without a hypothesis |
| `competitive` | Comparing options, threat assessment | Ranks, differentiates, recommends |
| `causal` | Debugging, incident analysis | Traces causes, identifies root factors |
| `deep` | Comprehensive research synthesis | Exhaustive, multi-angle, slower |

## The Recipe

### exploratory: discover what's in the data

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

# Analyze a namespace of accumulated facts
table = client.logic.tabulate(
    query="attribute(Entity, Attr, Value)",
    namespace="customer_feedback"
)

result = client.prism.analyze(
    goal="What are the dominant themes and any surprising patterns in this feedback?",
    data=table["cache_ref"],
    mode="exploratory"
)

print(result["result"])
# Returns: thematic clusters, outliers, confidence levels, suggested follow-ups
```

### competitive: rank, differentiate, recommend

```python
# After researching competitors into LC
table = client.logic.tabulate(
    query="attribute(Competitor, Attribute, Value)",
    namespace="competitive_intel"
)

result = client.prism.analyze(
    goal="""Produce a competitive threat matrix:
    1. Rank competitors by overlap with DataGrout's core use case
    2. For each, identify their key differentiator and main weakness
    3. Recommend two positioning moves for DataGrout""",
    data=table["cache_ref"],
    mode="competitive"
)

print(result["result"])
```

### causal: trace causes and effects

```python
# After asserting system observations into a forensic namespace
result = client.prism.analyze(
    goal="""Given these system observations, identify:
    1. The root cause of the 504 errors
    2. The causal chain from root cause to symptom
    3. Which component change would have the highest blast radius if fixed""",
    data=client.logic.tabulate(
        query="attribute(Component, Property, Value)",
        namespace="_forensic_myapp"
    )["cache_ref"],
    mode="causal"
)

print(result["result"])
```

### deep: exhaustive multi-angle synthesis

```python
# For high-stakes analysis where completeness matters more than speed
search = client.inference.rfi(
    topic="Enterprise adoption blockers for symbolic AI reasoning tools",
    sections=["Technical", "Organizational", "Economic", "Competitive"],
    quality="high"
)

result = client.prism.analyze(
    goal="""Produce a comprehensive assessment of adoption blockers:
    - Technical barriers: integration complexity, learning curve, tooling gaps
    - Organizational: change management, skills, champion dynamics
    - Economic: ROI proof, pricing sensitivity, budget cycles
    - Competitive: incumbent inertia, make-vs-buy calculus
    Conclude with a ranked list of the top 3 blockers by impact.""",
    data=search["cache_ref"],
    mode="deep"  # thorough but slower; auto-promotes to background for large data
)

print(result["result"])
```

## Using analyze in a pipeline

Analyze almost always sits at the end of a pipeline, consuming a `cache_ref`:

```python
# Full pipeline: search -> refract -> assert -> tabulate -> analyze
search = client.inference.search(
    query="SaaS churn benchmarks by company size 2026",
    quality="high"
)

facts = client.prism.refract(
    goal="Extract: company_size_tier, avg_churn_pct, industry, source_year",
    data=search["cache_ref"]
)

client.logic.assert_facts(facts=facts["result"], namespace="churn_benchmarks")

table = client.logic.tabulate(
    query="attribute(Company, Metric, Value)",
    namespace="churn_benchmarks"
)

analysis = client.prism.analyze(
    goal="How does churn vary by company size, and what's the benchmark for mid-market B2B SaaS?",
    data=table["cache_ref"],
    mode="exploratory"
)

print(analysis["result"])
```

## Structured output

Request a specific output format in your goal:

```python
result = client.prism.analyze(
    goal="""Return a JSON object with:
    {
      "threat_level": "low" | "medium" | "high",
      "top_risks": [{"risk": str, "likelihood": float, "impact": float}],
      "recommended_actions": [str],
      "confidence": float
    }""",
    data=data_ref,
    mode="competitive"
)

import json
parsed = json.loads(result["result"])
```

## Analyze vs refract

| Need | Use |
|------|-----|
| Normalize data to a consistent structure | `prism.refract` |
| Extract specific fields from free-form text | `prism.refract` |
| Summarize, interpret, draw conclusions | `prism.analyze` |
| Answer a question about data | `prism.analyze` |
| Rank, compare, or recommend | `prism.analyze` with `mode="competitive"` |
| Debug a system or incident | `prism.analyze` with `mode="causal"` |

## See also

- [refract-reshape](../../../patterns/refract-reshape/) — normalize data before analyzing
- [research-to-report](../../combined/research-to-report/) — search -> refract -> analyze pipeline
- [competitive-intel-pipeline](../../combined/competitive-intel-pipeline/) — analyze in competitive mode at scale
- [forensic/causal-chain-tracing](../../forensic/causal-chain-tracing/) — causal mode paired with Prolog logic
