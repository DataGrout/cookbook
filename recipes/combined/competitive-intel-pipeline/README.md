# Recipe: Competitive Intel Pipeline

## What it does

Research the top N competitors, store structured findings as Prolog facts, produce a threat assessment matrix.

Pipeline: `inference.search` -> `prism.refract` -> `logic.assert` -> `logic.tabulate` -> `frame.group` -> `prism.analyze`

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `inference.search` × N | 1–3 each | Grounded research per competitor |
| `prism.refract` × N | 1 each | Normalize to entity/attribute facts |
| `logic.assert` × N | 1 each | Persist facts in Prolog namespace |
| `logic.tabulate` | 0 | Query facts as flat table |
| `frame.group` | 0 | Group by threat level |
| `prism.analyze` | 1–3 | Final threat matrix |

## Credits cost

~15–25 credits for 5 competitors.

## The Recipe

```python
from datagrout_conduit import Client
import asyncio

client = Client(api_key="your-key")

competitors = [
    "LangChain",
    "LlamaIndex",
    "Weights & Biases",
    "Comet ML",
    "Arize AI"
]
namespace = "competitive_intel"
space = "AI observability and orchestration tools"

async def research_competitor(name: str) -> None:
    """Research one competitor and store structured facts."""
    print(f"Researching {name}...")

    # Step 1: Grounded research
    search = client.inference.search(
        query=f"{name} product capabilities pricing customers market position {space}",
        quality="high"
    )

    # Step 2: Normalize to structured facts
    facts = client.prism.refract(
        goal=f"""Extract a competitor profile as entity/attribute facts.
        Entity name: {name.lower().replace(' ', '_')}
        Attributes to extract:
        - strengths (list as separate facts)
        - weaknesses (list as separate facts)
        - pricing_model (free/freemium/paid/enterprise)
        - primary_use_case
        - target_customer (startup/smb/enterprise/all)
        - open_source (true/false)
        - threat_level (low/medium/high) based on competitive overlap with DataGrout
        - differentiator (what makes them unique)
        """,
        data=search["result"]
    )

    # Step 3: Persist
    client.logic.assert_facts(
        facts=facts["result"],
        namespace=namespace
    )
    print(f"  {name}: {len(facts['result'])} facts stored")

# Research all competitors (in parallel where possible)
for competitor in competitors:
    research_competitor(competitor)

print("\nAll research complete. Building assessment matrix...")

# Step 4: Flatten stored facts to table
table = client.logic.tabulate(
    query="attribute(Competitor, Attribute, Value)",
    namespace=namespace,
    columns=["Competitor", "Attribute", "Value"]
)

# Step 5: Group by threat level
threat_groups = client.frame.group(
    input=table["cache_ref"],
    by="threat_level",
    agg={"Competitor": "list"}
)

# Step 6: Final analysis
assessment = client.prism.analyze(
    goal="""Produce a threat assessment matrix:
    1. Rank competitors by threat level (high/medium/low)
    2. For each, identify: key differentiator, overlap areas, mitigation strategy
    3. Identify any 2-3 partnership opportunities among low-threat competitors
    4. Recommend top 2 competitive moves for DataGrout based on this landscape
    """,
    data=threat_groups["cache_ref"],
    mode="competitive"
)

print(assessment["result"])
```

## Cross-session queryability

After running this pipeline, the facts persist across sessions:

```python
# Next session: no re-research needed
result = client.logic.query(
    query="attribute(Competitor, threat_level, high)",
    namespace="competitive_intel"
)
# Returns all high-threat competitors from last run

# Add new findings without replacing old ones
client.logic.assert_facts(
    facts=[{"type": "attribute", "entity": "newco", "attribute": "threat_level", "value": "high"}],
    namespace="competitive_intel"
)
```

## Try it with Claude Code

Run this pipeline via natural language using the DataGrout MCP integration:

1. Set up MCP: `claude mcp add datagrout -- npx -y @datagrout/mcp`
2. Ask Claude Code: "Research the top 5 competitors to DataGrout and produce a threat matrix using DG tools"
3. Claude will execute this pipeline using the MCP tools

## Variations

**Add citation tracking:**
```python
# Include source URLs in the facts
client.logic.assert_facts(facts=[{
    "type": "attribute",
    "entity": competitor.lower(),
    "attribute": "source_url",
    "value": search["citations"][0]["url"]
}], namespace=namespace)
```

**Scheduled refresh:**
```python
# Re-run research monthly with scheduler
client.governor.schedule(
    prompt="Re-research all competitors in competitive_intel namespace",
    schedule="0 9 1 * *"  # 9am on the 1st of each month
)
```

**Export to report:**
```python
# After analysis, save to a doc
client.docs.create(
    title=f"Competitive Assessment — {datetime.now().strftime('%Y-%m')}",
    body=assessment["result"],
    tags=["competitive", "quarterly"]
)
```

## See also

- [benchmarks/05-competitive-research](https://github.com/DataGrout/benchmarks) — benchmark using this pattern
- [research-to-report](../research-to-report/) — research -> structured report pipeline
