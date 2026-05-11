# Recipe: Concept Exploration

## What it does

The latent suite surfaces relationships and orientations that aren't explicit in your data. `latent.orient` positions a concept in a multi-dimensional semantic space. `latent.horizon` finds what's just beyond the edge of a known concept cluster, useful for discovering adjacent ideas, blind spots, and emerging themes.

## Tools used

| Tool | Credits | Purpose |
|------|---------|---------|
| `latent.orient` | 1–2 | Position a concept in semantic space relative to anchors |
| `latent.horizon` | 2–3 | Find concepts at the boundary of a known cluster |

## Credits cost

~1–3 credits per call. Results are cached, repeat queries on the same concept are free.

## The Recipe

### latent.orient: where does a concept sit?

Orient answers: "given these reference points, where does this concept land?" It returns a position vector and the nearest neighbors in each dimension.

```python
from datagrout_conduit import Client

client = Client(api_key="your-key")

result = client.latent.orient(
    concept="DataGrout Logic Cell",
    axes=[
        {"label": "abstraction", "low": "concrete implementation", "high": "abstract interface"},
        {"label": "audience", "low": "developer tool", "high": "business user tool"},
        {"label": "novelty", "low": "established pattern", "high": "new paradigm"}
    ]
)

print(result["position"])
# {"abstraction": 0.72, "audience": 0.31, "novelty": 0.68}
# -> abstract interface, developer-leaning, fairly novel

print(result["nearest"])
# concepts closest to this position in the latent space
```

### latent.horizon: what's adjacent?

Horizon answers: "given what I know, what am I not seeing?" It takes a set of known concepts and finds what sits just beyond the boundary.

```python
result = client.latent.horizon(
    known=[
        "Prolog reasoning",
        "vector databases",
        "knowledge graphs",
        "symbolic AI"
    ],
    domain="AI memory and retrieval",
    count=5
)

for concept in result["horizon"]:
    print(f"{concept['label']}: {concept['description']}")
    print(f"  Distance from known cluster: {concept['distance']:.2f}")
    print(f"  Why it's at the boundary: {concept['rationale']}")
    print()

# Example output:
# Probabilistic logic programming: ...
# Neuro-symbolic hybrid models: ...
# Reactive Prolog: ...
```

## Use cases

### Competitive blind spot detection

```python
# What competitors are you not tracking?
result = client.latent.horizon(
    known=["LangChain", "LlamaIndex", "Weights & Biases", "Comet ML"],
    domain="AI orchestration and observability tooling",
    count=8
)

# Assert findings into LC for follow-up research
client.logic.assert_facts(facts=[
    {"type": "entity", "name": concept["label"].lower().replace(" ", "_"),
     "attributes": {"discovered_via": "latent.horizon", "distance": concept["distance"]}}
    for concept in result["horizon"]
], namespace="competitive_intel")
```

### Positioning a new feature

```python
# Where does a proposed feature sit relative to existing ones?
result = client.latent.orient(
    concept="natural language Prolog query interface",
    axes=[
        {"label": "user_effort", "low": "zero setup", "high": "expert configuration"},
        {"label": "power", "low": "limited queries", "high": "full logic programming"},
        {"label": "market_fit", "low": "niche", "high": "mass market"}
    ],
    anchors=[
        "SQL query editor",
        "ChatGPT conversation",
        "Datalog interface",
        "Wolfram Alpha"
    ]
)

print(result["position"])
print(result["nearest_anchors"])
```

### Exploring a research space

```python
# Map out what you know, find what you're missing
known_topics = [
    "transformer attention mechanisms",
    "RLHF training",
    "chain of thought prompting",
    "tool use in LLMs"
]

result = client.latent.horizon(
    known=known_topics,
    domain="LLM capability research",
    count=6
)

# Pipe to inference for follow-up research on each horizon concept
for concept in result["horizon"]:
    search = client.inference.search(
        query=f"{concept['label']} research papers and applications 2025 2026",
        quality="standard"
    )
    client.logic.assert_facts(facts=[
        {"type": "attribute", "entity": concept["label"],
         "attribute": "research_summary", "value": search["result"][:500]}
    ], namespace="research_map")
```

## Combining orient and horizon

```python
# Step 1: Orient your product to understand its current position
position = client.latent.orient(
    concept="DataGrout",
    axes=[
        {"label": "technical_depth", "low": "no-code", "high": "developer-first"},
        {"label": "scope", "low": "single use case", "high": "general platform"}
    ]
)

# Step 2: Find what's at the horizon from that position
horizon = client.latent.horizon(
    known=["DataGrout", "LangChain", "Weights & Biases"],
    domain="AI developer tooling",
    count=5
)

# Step 3: Analyze the strategic picture
analysis = client.prism.analyze(
    goal="""Given our current position and the horizon concepts found,
    identify: (1) the most relevant adjacent market, (2) the biggest
    blind spot in our current positioning, (3) one strategic move.""",
    data={"position": position["position"], "horizon": horizon["horizon"]}
)
print(analysis["result"])
```

## See also

- [competitive-intel-pipeline](../../combined/competitive-intel-pipeline/) — research pipeline that can feed latent.horizon
- [refract-reshape](../../../patterns/refract-reshape/) — normalize findings before piping to latent tools
- [advanced/forensic-rules-library](../../../advanced/forensic-rules-library/) — pair with logic to reason over discovered concepts
